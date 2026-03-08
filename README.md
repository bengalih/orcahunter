# orcahunter

**Hunt down the effective settings in, and between, your OrcaSlicer profiles.**

The effective print profile in OrcaSlicer is created by combining three settings sections: machine, filament, and process. Each section contributes its own group of settings, and where sections define conflicting settings, a priority rule determines which one wins.

Each of these three sections is not a single flat file. Each one is itself built from an inheritance chain — a file that inherits from a parent, which inherits from a grandparent, all the way back to a vendor or common base. Settings can be introduced or overridden at any level of that chain. By the time you are looking at your specific filament profile, the value actually in effect for any given setting may have originated several levels up the chain, then been partially overridden by an intermediate file, then overridden again by a cross-section priority rule.

`orcahunter` resolves and reports these settings — walking every inheritance chain for every section, applying all override rules, and presenting a single unified view of what every setting actually is and exactly where it came from. In most cases it is not necessary to know the full line of inheritance, but a complete output of effective settings can be helpful for review.

`orcahunter` shows its real utility when comparing two print profiles (or parts of print profiles) to each other. By building the full inheritance chains and processing filament overrides for both sides, it shows all the settings that diverge between them. This is useful for tracking down setting mismatches or understanding why one profile produces better or worse results than another.

---

## Table of Contents

- [Requirements ⚙️](#requirements)
- [Installation 🚀](#installation)
- [How OrcaSlicer Profiles Work 🖨️](#how-orcaslicer-profiles-work)
- [How orcahunter Works 🔍](#how-orcahunter-works)
  - [Inheritance Resolution](#inheritance-resolution)
  - [Section Priority](#section-priority)
  - [Filament Override Keys](#filament-override-keys)
  - [Override Flags](#override-flags)
- [Configuration File 🗂️](#configuration-file)
- [Usage 💻](#usage)
  - [Specifying Profiles](#specifying-profiles)
  - [Specifying Search Directories](#specifying-search-directories)
  - [All Options](#all-options)
- [Output Format 📄](#output-format)
  - [Single Profile Mode](#single-profile-mode)
  - [Diff Mode](#diff-mode)
- [Examples 💡](#examples)
- [Color Customization 🎨](#color-customization)
- [Extending the Config 🔧](#extending-the-config)

---

## Requirements

- Python 3.10 or later

---

## Installation

1. Copy `orcahunter.py` and `orcahunter_config.json` to the same directory.
2. Make it executable (Linux/macOS):
   ```bash
   chmod +x orcahunter.py
   ```
3. Run it:
   ```bash
   python3 orcahunter.py --help
   ```

Both files must always remain in the same directory. `orcahunter` locates its config by looking alongside itself.

---

## How OrcaSlicer Profiles Work

OrcaSlicer profiles are organized into three **sections**, each stored as a JSON file:

| Section    | Controls                                         |
|------------|--------------------------------------------------|
| `machine`  | Printer hardware: bed size, speeds, gcode        |
| `filament` | Material properties: temps, cooling, retraction  |
| `process`  | Print quality: layer height, infill, supports    |

Each profile file can declare `"inherits": "parent_profile_name"`, creating a chain from a general base up to the specific profile you selected. Settings defined closer to the child take priority — a value in your specific profile always wins over the same value in an ancestor.

A typical chain might look like:

```
fdm_machine_base.json               ← root (most general)
  └── fdm_machine_common.json
        └── fdm_vendor_common.json
              └── My Printer.json   ← child (most specific, wins)
```

A setting defined in `My Printer.json` overrides the same setting in any ancestor.

---

## How orcahunter Works

### Inheritance Resolution

For each profile file you provide, `orcahunter` walks the full `inherits` chain, loading every ancestor. It builds a complete picture of every setting at every level, then determines the **effective value** — the value that OrcaSlicer would actually use.

Inheritance is built by OrcaSlicer in the background and the results are what is displayed in the GUI. When you modify a setting, it is saved to your particular profile, which applies last in the chain of inheritance and overrides any conflicting values set before it. This design exists so OrcaSlicer can set some values common to all printers, others specific to a particular vendor, and still others to a model. The final set of user settings is applied on top of those to ultimately produce what the GUI displays. While knowing the full provenance of a setting in the inheritance chain is not usually necessary, it is sometimes helpful for troubleshooting, building, or porting print profiles between OrcaSlicer versions or forks.

**Locality-first search:** When resolving an `inherits` reference, `orcahunter` first searches up the directory tree from the file that declared the inheritance, before falling back to your global `--search-dir` paths. This ensures that vendor-specific files (e.g. a vendor's `fdm_machine_common.json`) always resolve to their own vendor's copy, not another vendor's copy with the same filename.

**Backup path skipping:** Any file whose path contains a directory component with `backup` in the name (case-insensitive) is automatically skipped during search.

### Section Priority

For most settings, when the same key appears in multiple sections, the higher-priority section wins:

```
process  >  filament  >  machine
```

However, this is not a fixed global rule. OrcaSlicer introduced a mechanism where certain filament-section keys override settings from specific sections — and that target section varies by key. See [Filament Override Keys](#filament-override-keys) below for the full picture.

A section override is always shown in the Override Detail section and counted under `Section overrides`.

### Filament Override Keys

OrcaSlicer uses a special mechanism where certain filament-section keys can override settings that nominally belong to another section. These keys carry a `filament_` prefix and are always resolved against their base key name in the output.

For example: `filament_retract_before_wipe` set in `filament` will override `retract_before_wipe` set in `machine`.

There are two categories, and they differ in which section they override:

**Filament overrides machine** — the original mechanism. Retraction and toolchange settings like `filament_retraction_speed` override `retraction_speed` from the machine section. The effective priority for these keys is:

```
process  >  filament  >  machine
```
_(filament wins over machine, process still wins over filament)_

**Filament overrides process** — as of [OrcaSlicer 2.3.2 RC2](https://github.com/OrcaSlicer/OrcaSlicer/pull/11194), filament profiles also support ironing overrides of process settings. Ironing settings like `filament_ironing_flow` override `ironing_flow` from the process section, allowing per-filament ironing tuning. The effective priority for these keys is:

```
filament  >  process  >  machine
```
_(filament wins over process and is the highest priority for these settings)_

In both cases `orcahunter` handles this transparently:
- The `filament_` prefixed key **never appears** in the output effective values (but does appear in the override detail).
- Its value is shown under the base key name (e.g. `retraction_speed`, `ironing_flow`).
- If the value actually differs from the target section's value, a `(*)` marker appears and the `filament_ prefix` sub-count in Counts reflects it.
- The base key is derived by stripping the `filament_` prefix, except for a small number of irregular keys (e.g. `activate_chamber_temp_control`, `idle_temperature`) which are listed in `filament_base_key_exceptions` in the config.

The full list of known override keys and their targets is maintained in `orcahunter_config.json`.

### Override Flags

Each setting in the output is classified at display time:

| Flag                 | Meaning                                                                                      |
|----------------------|----------------------------------------------------------------------------------------------|
| **Section override** | A higher-priority section set a different value than a lower-priority one                    |
| **Filament prefix**  | A `filament_xxx` key overrode its base key in machine or process with a different value      |
| **Inherit override** | The value changed within the winning section's own inheritance chain (only shown with `--show-inheritance`) |

Settings with any override are marked with `(*)` in the Effective Values listing.

---

## Configuration File

`orcahunter_config.json` must live alongside `orcahunter.py`. It controls two things:

### `meta_keys`

Keys that are structural/administrative and should never appear in the output. These are OrcaSlicer internals that are not print settings:

```json
"meta_keys": [
    "type", "name", "inherits", "from", "setting_id", "instantiation",
    "version", "is_custom_defined", "base_id", "filament_id",
    "filament_settings_id", "print_settings_id", "printer_settings_id",
    "compatible_printers", "compatible_printers_condition",
    "compatible_prints", "compatible_prints_condition"
]
```

### `known_filament_overrides`

Groups `filament_` override keys by the section they target. This is how `orcahunter` knows both the base key name and the correct priority order to apply for each filament override key.

```json
"known_filament_overrides": {
    "machine": [
        "filament_retraction_speed",
        "filament_retraction_length",
        "filament_z_hop",
        "filament_wipe",
        ...
    ],
    "process": [
        "filament_ironing_flow",
        "filament_ironing_spacing",
        "filament_ironing_inset",
        "filament_ironing_speed"
    ]
}
```

Keys listed under `"machine"` override machine settings — filament wins over machine but loses to process. Keys listed under `"process"` override process settings — filament wins over process and is the highest priority for that setting.

Keys beginning with `_` are treated as comments and ignored.

### `filament_base_key_exceptions`

For the small number of filament override keys whose base key cannot be derived by simply stripping the `filament_` prefix, an explicit mapping is provided here:

```json
"filament_base_key_exceptions": {
    "activate_chamber_temp_control": "activate_chamber_temp_control",
    "idle_temperature":              "idle_temperature"
}
```

### Versioning

The config file has a `"version"` integer field. `orcahunter` will refuse to run if the config version is older than it requires, ensuring compatibility.

### Updating

The config JSON can be edited by the end user to add or change overrides. This should be done with caution and should be based on the overrides defined in the OrcaSlicer source (`/src/libslic3r/Preset.cpp`). See [Extending the Config](#extending-the-config) for more information.

---

## Usage

```
orcahunter.py [-h] [-b FILE] [--base-machine FILE] [--base-filament FILE]
              [--base-process FILE] [-d FILE] [--diff-machine FILE]
              [--diff-filament FILE] [--diff-process FILE]
              [-s DIR] [-S DIR] [-f TEXT] [-o]
              [--show-inheritance] [--full-gcode] [--no-color]
              [--output FILE] [--ignore-missing]
```

### Specifying Profiles

There are two ways to specify profile files:

**1. Inferred role (`-b` / `-d`):**  
The section role (machine / filament / process) is inferred from the file's **parent directory name**. This works naturally with standard OrcaSlicer profile directory layouts.

```bash
orcahunter.py \
  -b /profiles/machine/My\ Printer.json \
  -b /profiles/filament/Generic\ PLA.json \
  -b /profiles/process/0.2mm\ Standard.json \
  -s /profiles
```

**2. Explicit role (`--base-machine`, `--base-filament`, `--base-process`):**  
Use these when the file is not inside a directory named `machine`, `filament`, or `process`.

```bash
orcahunter.py \
  --base-machine "My Printer.json" \
  --base-filament "Generic PLA.json" \
  --base-process "0.2mm Standard.json" \
  -s /profiles
```

You do not need to provide all three sections. Any combination is valid — provide only the sections you care about.

### Specifying Search Directories

`orcahunter` needs to find inherited parent files. Use `-s` to tell it where to search:

```bash
orcahunter.py -b filament/PLA.json -s /path/to/orcaslicer/system -s /path/to/custom/profiles
```

- Search is **recursive** — subdirectories are included automatically.
- Multiple `-s` flags are allowed and searched in order.
- Profile file parent directories are **always** searched automatically, even without `-s`.
- If a filename appears in more than one location (excluding backup paths), `orcahunter` will error and list the duplicates so you can resolve the ambiguity.

### All Options

#### Baseline Profile

| Flag | Long form | Description |
|------|-----------|-------------|
| `-b FILE` | `--base FILE` | Baseline profile JSON. Role inferred from parent dir. Repeatable. |
| | `--base-machine FILE` | Baseline machine profile (explicit role). |
| | `--base-filament FILE` | Baseline filament profile (explicit role). |
| | `--base-process FILE` | Baseline process profile (explicit role). |

#### Diff Profile

Omitted diff roles automatically fall back to the corresponding baseline profile, so you only need to specify what differs.

| Flag | Long form | Description |
|------|-----------|-------------|
| `-d FILE` | `--diff FILE` | Diff profile JSON. Role inferred from parent dir. Repeatable. |
| | `--diff-machine FILE` | Diff machine profile (explicit role). |
| | `--diff-filament FILE` | Diff filament profile (explicit role). |
| | `--diff-process FILE` | Diff process profile (explicit role). |

#### Search Paths

| Flag | Long form | Description |
|------|-----------|-------------|
| `-s DIR` | `--search-dir DIR` | Baseline search dir (recursive). Repeatable. |
| `-S DIR` | `--diff-search-dir DIR` | Diff search dir (recursive). Repeatable. Defaults to `-s` dirs if omitted. |

#### Output Options

| Flag | Long form | Description |
|------|-----------|-------------|
| `-f TEXT` | `--filter TEXT` | Only show keys whose name contains TEXT (case-insensitive). |
| `-o` | `--overrides-only` | Show only overridden settings. Single-profile mode only. |
| | `--show-inheritance` | Expand override detail to show the full per-file inheritance chain. |
| | `--full-gcode` | Show full gcode values untruncated (default: truncate at 150 chars). |
| | `--no-color` | Disable ANSI color output. |
| | `--output FILE` | Write output to FILE instead of stdout (implies `--no-color`). |
| | `--ignore-missing` | Warn and skip inherited files that cannot be found, rather than erroring. |

---

## Output Format

### Single Profile Mode

Output is divided into three sections:

#### Header

Shows the full resolved inheritance chain for each section, child-first:

```
==============================================================================
  OrcaSlicer Effective Settings
==============================================================================
  MACHINE
    /profiles/machine/My Printer.json  -->
    /profiles/system/fdm_vendor_common.json  -->
    /profiles/system/fdm_machine_common.json  -->
    /profiles/system/fdm_machine_base.json
  FILAMENT
    /profiles/filament/Generic PLA.json  -->
    /profiles/system/fdm_filament_pla_common.json  -->
    /profiles/system/fdm_filament_base.json
  PROCESS
    /profiles/process/0.2mm Standard.json  -->
    /profiles/system/fdm_process_base.json
==============================================================================
```

#### Effective Values

Every setting with its winning value. Settings marked `(*)` were arrived at via one or more overrides:

```
  Effective Values  (* = overridden)
  ----------------------------------------------------------------------------

  bed_temperature: 35
  cool_plate_temp: 35
  cool_plate_temp_initial_layer: 35
  default_filament_colour: []
  enable_overhang_bridge_fan: 1  (*)
  fan_cooling_layer_time: 100
  fan_max_speed: 100  (*)
  retraction_speed: 45  (*)
  travel_speed: 250  (*)
```

#### Override Detail

Every overridden setting showing each contributing value, lowest priority first. The last (winning) entry is highlighted:

```
  Override Detail
  ----------------------------------------------------------------------------
  Lowest priority first. Last entry is the effective value.
  ----------------------------------------------------------------------------

  retraction_speed: 45
    [40]  (fdm_machine_common.json)  [retraction_speed]
    [50]  (fdm_vendor_common.json)  [retraction_speed]
    [45]  (Generic PLA.json)  [filament_retraction_speed]

  travel_speed: 250
    [200]  (fdm_machine_common.json)  [travel_speed]
    [250]  (My Printer.json)  [travel_speed]
```

With `--show-inheritance`, every file in the full inheritance chain is shown rather than just the tip of each section.

#### Counts

```
==============================================================================
  Counts
==============================================================================
  Total settings    : 56
  No override       : 50
  Section overrides : 6
    filament_ prefix  : 1
```

With `--show-inheritance`, an additional line appears:

```
==============================================================================
  Counts
==============================================================================
  Total settings    : 56
  No override       : 40
  Section overrides : 6
    filament_ prefix  : 1
  Inherit overrides : 10
```

### Diff Mode

Diff mode is triggered whenever any `-d` / `--diff-*` flag is provided. Output shows three categorized sections followed by a summary:

```
==============================================================================
  OrcaSlicer Profile Diff
==============================================================================
  BASELINE
    FILAMENT
      /profiles/filament/Generic PLA.json  -->
      /profiles/system/fdm_filament_base.json
  DIFF
    FILAMENT
      /profiles/filament/Generic PETG.json  -->
      /profiles/system/fdm_filament_petg_common.json  -->
      /profiles/system/fdm_filament_base.json
==============================================================================

  Changed  (4)
  ----------------------------------------------------------------------------
  bed_temperature:  35  ->  70
  fan_max_speed:  100  ->  60
  filament_max_volumetric_speed:  21  ->  14
  nozzle_temperature:  220  ->  240

  Only in baseline  (2)
  ----------------------------------------------------------------------------
  cool_plate_temp: 35
  cool_plate_temp_initial_layer: 35

  Only in diff  (1)
  ----------------------------------------------------------------------------
  eng_plate_temp: 70

==============================================================================
  Summary
==============================================================================
  Changed          : 4
  Only in baseline : 2
  Only in diff     : 1
  Identical        : 49
```

---

## Examples

### View the full effective settings for a printer/filament/process combination

```bash
python3 orcahunter.py \
  -b "machine/My Printer 0.4 nozzle.json" \
  -b "filament/Generic PLA.json" \
  -b "process/0.2mm Standard.json" \
  -s ~/Library/Application\ Support/OrcaSlicer/system \
  -s ~/Library/Application\ Support/OrcaSlicer/user
```

### View only settings that are overridden (no-noise mode)

```bash
python3 orcahunter.py \
  -b "machine/My Printer.json" \
  -b "filament/Generic PLA.json" \
  -b "process/0.2mm.json" \
  -s /profiles \
  --overrides-only
```

### Search for all retraction-related settings

```bash
python3 orcahunter.py \
  -b "machine/My Printer.json" \
  -b "filament/Generic PLA.json" \
  -s /profiles \
  --filter retraction
```

### Show the full inheritance chain detail for overridden settings

```bash
python3 orcahunter.py \
  -b "machine/My Printer.json" \
  -b "filament/Generic PLA.json" \
  -b "process/0.2mm.json" \
  -s /profiles \
  --show-inheritance --overrides-only
```

### Diff two filament profiles against the same machine and process

Machine and process are shared from baseline — only specify what changes:

```bash
python3 orcahunter.py \
  -b "machine/My Printer.json" \
  -b "filament/Generic PLA.json" \
  -b "process/0.2mm Standard.json" \
  -d "filament/Generic PETG.json" \
  -s /profiles
```

### Diff a custom filament against a system filament

```bash
python3 orcahunter.py \
  --base-filament "Generic PLA.json" \
  --diff-filament "My Custom PLA.json" \
  -s ~/OrcaSlicer/system \
  -s ~/OrcaSlicer/user
```

### Diff two complete profiles — different machine and filament

```bash
python3 orcahunter.py \
  -b "machine/Printer A.json" \
  -b "filament/Generic PLA.json" \
  -b "process/0.2mm.json" \
  -d "machine/Printer B.json" \
  -d "filament/Generic PLA.json" \
  -s /profiles
```

### Save output to a file for archiving or sharing

```bash
python3 orcahunter.py \
  -b "machine/My Printer.json" \
  -b "filament/Generic PLA.json" \
  -b "process/0.2mm.json" \
  -s /profiles \
  --output my_profile_dump.txt
```

### Filter and save only retraction settings to a file

```bash
python3 orcahunter.py \
  -b "machine/My Printer.json" \
  -b "filament/Generic PLA.json" \
  -s /profiles \
  --filter retraction \
  --output retraction_settings.txt
```

### Explicit roles (files not in standard directory layout)

```bash
python3 orcahunter.py \
  --base-machine "My Printer.json" \
  --base-filament "Generic PLA.json" \
  --base-process "0.2mm Standard.json" \
  -s /profiles/system \
  -s /profiles/custom
```

### Skip inherited files that cannot be found

```bash
python3 orcahunter.py \
  -b "filament/Experimental PLA.json" \
  -s /profiles \
  --ignore-missing
```

### Use separate search directories for baseline and diff

Useful when comparing profiles from two different OrcaSlicer installations or versions:

```bash
python3 orcahunter.py \
  -b "machine/My Printer.json" \
  -b "filament/Generic PLA.json" \
  -s /orcaslicer_v1/system \
  -d "filament/Generic PLA.json" \
  -S /orcaslicer_v2/system
```

---

## Color Customization

Colors are defined as constants near the top of `orcahunter.py` and can be changed without touching any logic:

```python
COLOR_KEY      = COLORS["bright_white"]   # setting key names
COLOR_VALUE    = COLORS["cyan"]           # effective values
COLOR_MARKER   = COLORS["bright_yellow"]  # (*) override marker
COLOR_MACHINE  = COLORS["blue"]           # machine section labels
COLOR_FILAMENT = COLORS["green"]          # filament section labels
COLOR_PROCESS  = COLORS["yellow"]         # process section labels
COLOR_SOURCE   = COLORS["dim"]            # source filenames
COLOR_HEADER   = COLORS["bold"]           # section headers
```

All standard ANSI colors are available in the `COLORS` dict: `black`, `red`, `green`, `yellow`, `blue`, `magenta`, `cyan`, `white`, and their `bright_` variants, plus `bold` and `dim`.

Color output is automatically disabled when writing to a file (`--output`) or when stdout is not a terminal. Use `--no-color` to disable it explicitly.

---

## Extending the Config

### Adding new filament override keys

As OrcaSlicer adds new `filament_` override keys in future versions, add them to the appropriate group in `orcahunter_config.json`. The group name is the section they override — `"machine"` or `"process"`. The correct grouping can be verified in the OrcaSlicer source at `/src/libslic3r/Preset.cpp`.

```json
"known_filament_overrides": {
    "machine": [
        "filament_new_retraction_key",
        ...
    ],
    "process": [
        "filament_new_ironing_key",
        ...
    ]
}
```

The base key is derived automatically by stripping the `filament_` prefix. If the key does not follow that pattern, add an explicit mapping to `filament_base_key_exceptions`:

```json
"filament_base_key_exceptions": {
    "irregular_filament_key": "its_base_key"
}
```

### Adding new meta keys

If OrcaSlicer adds new structural/administrative keys that should not appear in output, add them to `meta_keys`:

```json
"meta_keys": [
    "type", "name", "inherits",
    "new_structural_key",
    ...
]
```

### Config versioning

The `"version"` field in `orcahunter_config.json` allows the script to detect incompatible config files. If you make breaking changes to the config structure, increment the version and update `CONFIG_MIN_VERSION` in `orcahunter.py` accordingly.
