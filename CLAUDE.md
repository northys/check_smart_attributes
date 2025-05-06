# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Check_smart_attributes is a Nagios/Icinga plugin that monitors SMART attributes of hard drives and SSDs. It allows system administrators to proactively detect potential storage device failures by monitoring various SMART attributes.

## Usage Commands

Basic command structure:
```bash
sudo ./check_smart_attributes -dbj ./check_smartdb.json -d /dev/sda
```

Common usage patterns:

1. Check a single device:
```bash
sudo ./check_smart_attributes -dbj ./check_smartdb.json -d /dev/sda
```

2. Check multiple devices:
```bash
sudo ./check_smart_attributes -dbj ./check_smartdb.json -d /dev/sda -d /dev/sdb
```

3. Check all NVMe devices using regex:
```bash
sudo ./check_smart_attributes -dbj ./check_smartdb.json -r '/dev/nvme[0-9]+'
```

4. Check with user configuration overrides:
```bash
sudo ./check_smart_attributes -dbj ./check_smartdb.json -d /dev/sda -ucfgj ./check_smartcfg.json
```

5. Check devices behind RAID controllers:
   - MegaRAID: 
     ```bash
     sudo ./check_smart_attributes -dbj ./check_smartdb.json -d megaraid4,/dev/sda
     ```
   - Adaptec:
     ```bash
     sudo ./check_smart_attributes -dbj ./check_smartdb.json -d /dev/sg2 -O sat
     ```
   - HP controllers (hpsa/cciss):
     ```bash
     sudo ./check_smart_attributes -dbj ./check_smartdb.json -d cciss,1_/dev/sg2
     ```
   - 3ware:
     ```bash
     sudo ./check_smart_attributes -dbj ./check_smartdb.json -d 3ware,8,/dev/twa0
     ```

## How It Works

1. The plugin uses the `smartctl` command from the smartmontools package to retrieve SMART attribute data from storage devices
2. Device attributes are matched against values in the `check_smartdb.json` file, which contains:
   - Device model mappings
   - SMART attribute ID mappings (VALUE or RAW_VALUE)
   - Threshold configurations for warnings/critical alerts
   - Performance data configuration

3. The plugin can be customized with a user config file `check_smartcfg.json` to override default thresholds

4. Output follows the Nagios/Icinga plugin format, with performance data for integration with monitoring systems

## Codebase Structure

The main components are:

1. `check_smart_attributes` - Perl script that implements the plugin
2. `check_smartdb.json` - Database of device models and their SMART attributes
3. `check_smartcfg.json` - User configuration file to override default settings

### JSON Configuration Structure

The `check_smartdb.json` file follows this structure:
```json
{
  "Devices": {
    "Device Model Name": {
      "Device": ["Manufacturer Model Name"],
      "ID#": {
        "1": {"value": "VALUE", "comment": "Raw Read Error Rate"},
        "5": {"value": "RAW_VALUE", "comment": "Re-allocated Sector Count"}
      },
      "Threshs": {
        "1": ["62:", "52:"],
        "5": ["20", "40"]
      },
      "Perfs": ["194"]
    }
  }
}
```

When adding a new device:
1. The "Device" key must match the "Device Model" from smartctl output
2. The "ID#" section defines which values to use (VALUE or RAW_VALUE)
3. "Threshs" defines warning/critical thresholds for monitored attributes
4. "Perfs" specifies which attributes to include in performance data

Special devices:
- NVMe devices use a generic entry with specific parsing
- SAS devices use a generic entry with specific parsing

## Development Patterns

When adding new device support:
1. Find the device in the smartmontools database (`smartctl -a /dev/device | grep "Device is"`)
2. Add a new entry in `check_smartdb.json` with the correct device model name
3. Configure the appropriate SMART attribute IDs to monitor (VALUE or RAW_VALUE)
4. Set appropriate thresholds for warning/critical states
5. Define performance data attributes to report

When modifying the plugin:
1. Ensure the Perl requirements are met: Getopt::Long, JSON, Storable qw(dclone)
2. The plugin follows Nagios/Icinga plugin standards (exit codes STATE_OK, STATE_WARNING, etc.)
3. Any new device types may need special handling in the `getSmartctl` and `parseSmartctlOut` functions

## Database Structure Reference

The `check_smartdb.json` database organizes devices by category/family. Common entries include:

### Common Samsung SSD Entries:

- **Samsung PM893**: Line ~579
  - Models: MZ7L3240HCHQ-00A07, MZ7L3480HCHQ-00A07, MZ7L3960HCJR-00A07, MZ7L3960HBLT-00A07, etc.
  - Key attributes: Wear Leveling Count (177), Reserved Blocks (179/180), Temperature (194), etc.
  - Performance metrics: 177, 180, 190, 241, 242

- **Samsung PM883**: Similar structure to PM893
  - Models: MZ7LH240HAHQ-00005, MZ7LH480HAHQ-00005, etc.

### Database Entry Format:

```json
"Device Family Name": {
  "Device": ["MODEL1", "MODEL2", "MODEL3"],
  "ID#": {
    "5": {"value": "RAW_VALUE", "comment": "Re-allocated Sector Count"},
    "9": {"value": "RAW_VALUE", "comment": "Power On Hours Count"},
    "177": {"value": "VALUE", "comment": "Wear Leveling Count"},
    ...
  },
  "Threshs": {
    "5": ["10", "20"],       // Warning, Critical thresholds
    "177": ["11:", "6:"],    // ":": Greater is better (reverses comparison)
    ...
  },
  "Perfs": ["177", "180", "190", "241", "242"]  // Performance data to report
}
```

### Common Attributes to Monitor:

- 5: Reallocated Sector Count
- 9: Power-On Hours
- 12: Power Cycle Count
- 177: Wear Leveling Count (SSDs)
- 179/180: Reserved Block Counts (SSDs)
- 190/194: Temperature
- 197: Current Pending Sector Count
- 198: Offline Uncorrectable Sector Count
- 241/242: Total LBAs Written/Read (SSD wear indicators)

When adding a new device, first check if it belongs to an existing family; if so, add the model to the existing entry rather than creating a new one.

## Common Pitfalls and Issues

### Duplicate Device Family Names

The database must have unique "Device Family Name" entries. Having duplicate family names (e.g., two entries for "Seagate Exos X") will cause detection issues:

1. The plugin processes the database sequentially
2. When it finds the first matching family, it stops searching
3. If a device model is in a duplicate family entry that appears later in the file, it won't be detected

Example of problematic duplicate entries:
```json
"Devices": {
  "Seagate Exos X": {  // First entry will be used
    "Device": ["MODEL1", "MODEL2"]
  },
  // Many other entries...
  "Seagate Exos X": {  // This duplicate will be ignored!
    "Device": ["MODEL3", "MODEL4"]  // These models won't be detected
  }
}
```

**Always check for existing family names** before adding a new entry. If a family with the same name exists:
1. Add your new model to the existing family entry, or
2. Use a slightly different name for your new family (e.g., "Seagate Exos X16" instead of "Seagate Exos X")

### Model Matching Logic

The plugin uses Perl regex matching with the model string extracted from smartctl output:
```perl
if($currModel =~ /$_$/){
  $found=1;
}
```

Where:
- `$currModel` is a line like `Device Model:     ST6000NM019B-2TG103`
- `$_` is the model pattern from the database

This regex matching looks for an exact match at the end of the string (`$` anchor), so be precise with model strings.

## Working with the Database using `jq`

The `jq` tool is very useful for exploring the database structure and determining where a new device should be added. Below are practical examples for working with `check_smartdb.json`.

### Finding Device Entries

Search for existing device model patterns:

```bash
# Find all Samsung device entries
jq '.Devices | keys | .[]' check_smartdb.json | grep -i samsung

# Search for specific model patterns
jq '.Devices | keys | .[]' check_smartdb.json | grep -i "PM893\|PM883"
```

### Check if a Device Already Exists

Look for a specific device model string within the database:

```bash
# Find exact device model match
jq '.Devices | to_entries[] | select(.value.Device | index("SAMSUNG MZ7L3960HBLT-00A07"))' check_smartdb.json

# Find partial model match (useful for determining family)
jq '.Devices | to_entries[] | select(.value.Device | join("|") | test("MZ7L3"))' check_smartdb.json
```

### Examine Device Entry Structure

Inspect a specific device entry by family name:

```bash
# Get full device entry
jq '.Devices."Samsung PM893"' check_smartdb.json

# List models in a device family
jq '.Devices."Samsung PM893".Device' check_smartdb.json

# List attributes monitored for a device family
jq '.Devices."Samsung PM893"."ID#" | keys' check_smartdb.json

# List performance metrics for a device family
jq '.Devices."Samsung PM893".Perfs' check_smartdb.json
```

### Find Similar Devices

When adding a new device, find similar device entries to use as a template:

```bash
# Find device families with similar attribute patterns (e.g., SSD-specific attributes)
jq '.Devices | to_entries[] | select(.value."ID#" | has("177"))' check_smartdb.json

# Find device families monitoring similar metrics
jq '.Devices | to_entries[] | select(.value.Perfs | index("241"))' check_smartdb.json
```

### Practical Workflow for Adding a New Device

Example workflow for adding a Samsung MZ7L3960HBLT-00A07 SSD:

```bash
# 1. Look for existing device family matches
jq '.Devices | to_entries[] | select(.value.Device | join("|") | test("MZ7L3"))' check_smartdb.json

# 2. If found, check which family contains similar models
jq '.Devices | to_entries[] | select(.value.Device | join("|") | test("MZ7L3")) | .key' check_smartdb.json

# 3. Examine the specific device entry structure to follow the pattern
jq '.Devices."Samsung PM893"' check_smartdb.json

# 4. Update the device list in that family (done via text editor)
```

When working with `jq`, remember that the database has this hierarchical structure:
```
"Devices": {
  "Device Family Name": {
    "Device": [list of model strings],
    "ID#": {attribute mappings},
    "Threshs": {threshold settings},
    "Perfs": [performance metrics]
  }
}
```

This structure helps organize devices by family or type and ensures consistent monitoring across similar devices.