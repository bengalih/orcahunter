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
  - [Specifying Search Directories](#specifying-search-directories)
  - [Specifying Profiles](#specifying-profiles)
  - [Auto Mode](#auto-mode)
  - [All Options](#all-options)
- [Output Format 📄](#output-format)
  - [Single Profile Mode](#single-profile-mode)
  - [Diff Mode](#diff-mode)
- [Examples 💡](#examples)
- [Color Customization 🎨](#color-customization)
- [Extending the Config 🔧](#extending-the-config)
- [Limitations ⚠️](#limitations)

---

<a name="requirements"></a>

## ⚙️ Requirements

- Python 3.10 or later

---

<a name="installation"></a>

## 🚀 Installation

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

<a name="how-orcaslicer-profiles-work"></a>

## 🖨️ How OrcaSlicer Profiles Work

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

<a name="how-orcahunter-works"></a>

## 🔍 How orcahunter Works

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

| Flag                 | Marker | Meaning                                                                                      |
|----------------------|--------|----------------------------------------------------------------------------------------------|
| **Section override** | `(*)`  | A higher-priority section set a different value than a lower-priority one                    |
| **Filament prefix**  | `(*)`  | A `filament_xxx` key overrode its base key in machine or process with a different value      |
| **Inherit override** | `(*)`  | The value changed within the winning section's own inheritance chain (only shown with `--show-inheritance`) |
| **No base found**    | `(?)`  | A `filament_xxx` override key is set, but the corresponding base key does not appear in any inherited file — the value is real but cannot be confirmed as an override from the files alone |

Settings with a confirmed override are marked `(*)`. Settings where a filament override key is present but no base value exists to compare against are marked `(?)`.

---

<a name="configuration-file"></a>

## 🗂️ Configuration File

`orcahunter_config.json` must live alongside `orcahunter.py`. It controls output colors, structural key filtering, and filament override resolution.

### `colors`

Optional. Sets the color for each output role. If omitted entirely, built-in defaults are used. See [Color Customization](#color-customization) for the full role reference and valid color names.

```json
"colors": {
    "key":          "bright_white",
    "value":        "cyan",
    "marker":       "bright_yellow",
    "machine":      "blue",
    "filament":     "green",
    "process":      "yellow",
    "source":       "dim",
    "header":       "bold",
    "winner":       "bright_cyan",
    "loser":        "red",
    "diff_changed": "bright_yellow"
}
```

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

<a name="usage"></a>

## 💻 Usage

```
orcahunter.py [-h] [--auto] [-b FILE] [--base-machine FILE] [--base-filament FILE]
              [--base-process FILE] [-d FILE] [--diff-machine FILE]
              [--diff-filament FILE] [--diff-process FILE]
              [-s DIR] [-S DIR] [-f TEXT]
              [--overrides-only] [--values-only]
              [--show-inheritance] [--full-gcode] [--show-paths]
              [--color-code-values] [--no-color]
              [--ignore-type] [--diff-changed-only] [--conf FILE]
              [--ignore-missing] [--output FILE]
```

### Specifying Search Directories

Search directories are the foundation of how `orcahunter` locates files. They serve two purposes:

1. **Finding the profile files you specify** with `-b` and `-d` — you can pass just a filename rather than a full path, and `orcahunter` will find it within your search directories.
2. **Resolving inherited parent files** — as `orcahunter` walks each inheritance chain, it uses the search directories to locate every ancestor file.

Point your search directories at the root of your OrcaSlicer data — `orcahunter` searches recursively, so a single path covering both `system` and `user` profiles is usually sufficient.

**Linux / macOS:**
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer/system \
  -s ~/.config/OrcaSlicer/user \
  -b "My Printer.json" \
  -b "Generic PLA.json"
```

**Windows:**
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer\system" ^
  -s "%APPDATA%\OrcaSlicer\user" ^
  -b "My Printer.json" ^
  -b "Generic PLA.json"
```

Key behaviours:
- Search is **recursive** — all subdirectories are included automatically.
- Multiple `-s` flags are allowed and searched in order.
- The parent directory of each profile file you specify is **always** added to the search path automatically, even without `-s`.
- If the same filename is found in more than one location, `orcahunter` will error and list the duplicates so you can resolve the ambiguity. Paths containing `backup` in any directory component are automatically excluded from search.
- Use `-S` to specify separate search directories for diff profiles. If `-S` is omitted, the `-s` directories are used for both sides.

### Specifying Profiles

Profile files are specified with `-b` (baseline) and `-d` (diff). You can pass either a full path or just a filename — if just a filename is given, `orcahunter` searches for it across your `-s` directories.

There are two ways to tell `orcahunter` which section (machine / filament / process) a file belongs to:

**1. Inferred role (`-b` / `-d`):**
The section role is inferred from the file's **parent directory name**. This works naturally with standard OrcaSlicer profile directory layouts where files live inside folders named `machine`, `filament`, or `process`.

*Linux / macOS — full paths, role inferred from directory:*
```bash
python3 orcahunter.py \
  -b /profiles/machine/My\ Printer.json \
  -b /profiles/filament/Generic\ PLA.json \
  -b /profiles/process/0.2mm\ Standard.json \
  -s /profiles
```

*Windows — filenames only, role inferred from directory once found:*
```bat
python orcahunter.py ^
  -b "My Printer.json" ^
  -b "Generic PLA.json" ^
  -b "0.2mm Standard.json" ^
  -s "%APPDATA%\OrcaSlicer"
```

**2. Explicit role (`--base-machine`, `--base-filament`, `--base-process`):**
Use these when the file is not inside a directory named `machine`, `filament`, or `process` — for example, files copied to your Desktop or a working folder.

*Linux / macOS:*
```bash
python3 orcahunter.py \
  --base-machine "My Printer.json" \
  --base-filament "Generic PLA.json" \
  --base-process "0.2mm Standard.json" \
  -s /profiles
```

*Windows:*
```bat
python orcahunter.py ^
  --base-machine "My Printer.json" ^
  --base-filament "Generic PLA.json" ^
  --base-process "0.2mm Standard.json" ^
  -s "%APPDATA%\OrcaSlicer"
```

You do not need to provide all three sections. Any combination is valid — provide only the sections you care about.

### Auto Mode

`--auto` reads the currently active machine, filament, and process profile names directly from `orcaslicer.conf` — the same runtime state file OrcaSlicer writes when you have profiles selected in the GUI. This means you can run `orcahunter` without specifying any `-b` files at all, and it will resolve exactly what OrcaSlicer currently has loaded.

*Linux / macOS:*
```bash
python3 orcahunter.py --auto -s ~/.config/OrcaSlicer
```

*Windows:*
```bat
python orcahunter.py --auto -s "%APPDATA%\OrcaSlicer"
```

Key behaviours:
- `--auto` requires at least one `-s` search directory, which is where `orcaslicer.conf` is searched for.
- `--auto` and `-b` are **additive**: `-b` overrides `--auto` on a per-role basis. For example, `--auto -b "Other PLA.json"` uses the active machine and process from the conf, but substitutes the specified filament.
- The active bed type is read from the conf and displayed in the header.
- Profile names sourced from `--auto` are annotated with `(*)` in the chain header.
- By default `orcaslicer.conf` is searched by name within the `-s` directories. Use `--conf` to specify a different conf file by name or path.

**Using a custom conf file:**

If your conf file has a non-standard name or location, use `--conf`:

```bash
# Bare filename — searched within the -s directories
python3 orcahunter.py --auto --conf my_orca.conf -s ~/.config/OrcaSlicer

# Explicit path — used directly
python3 orcahunter.py --auto --conf /path/to/orcaslicer.conf -s ~/.config/OrcaSlicer
```

### All Options

#### Profile Input (Baseline)

| Flag | Description |
|------|-------------|
| `--auto` | Read active machine/filament/process from `orcaslicer.conf` in the `-s` dirs. Additive with `-b`. |
| `-b FILE` / `--base FILE` | Baseline profile JSON. Role inferred from parent dir name. Repeatable. |
| `--base-machine FILE` | Baseline machine profile (explicit role). |
| `--base-filament FILE` | Baseline filament profile (explicit role). |
| `--base-process FILE` | Baseline process profile (explicit role). |

#### Profile Input (Diff)

Omitted diff roles automatically fall back to the corresponding baseline profile, so you only need to specify what differs.

| Flag | Description |
|------|-------------|
| `-d FILE` / `--diff FILE` | Diff profile JSON. Role inferred from parent dir name. Repeatable. |
| `--diff-machine FILE` | Diff machine profile (explicit role). |
| `--diff-filament FILE` | Diff filament profile (explicit role). |
| `--diff-process FILE` | Diff process profile (explicit role). |

#### Search Paths

| Flag | Description |
|------|-------------|
| `-s DIR` / `--search-dir DIR` | Baseline search dir (recursive). Repeatable. Also used for `--auto` conf lookup. |
| `-S DIR` / `--diff-search-dir DIR` | Diff search dir (recursive). Repeatable. Defaults to `-s` dirs if omitted. |

#### Filtering

| Flag | Description |
|------|-------------|
| `-f TEXT` / `--filter TEXT` | Only show keys whose name contains TEXT (case-insensitive). |
| `--overrides-only` | Show only overridden (`*`) and unconfirmed (`?`) settings. Single-profile mode only. |
| `--values-only` | Show only the Effective Values section; suppress Override Detail and Counts. Pairs well with `-f` for quick targeted lookups. |

#### Display

| Flag | Description |
|------|-------------|
| `--show-inheritance` | Expand Override Detail to show every file in the inherit chain, not just the section tip. |
| `--full-gcode` | Show full gcode values untruncated (default: truncate at 75 chars). |
| `--show-paths` | Show full file paths in the chain header (default: filenames only). |
| `--color-code-values` | Color each key name by its winning section: machine=blue, filament=green, process=yellow. |
| `--no-color` | Disable ANSI color output. |

#### Comparison

| Flag | Description |
|------|-------------|
| `--ignore-type` | Treat a scalar and a single-element array as equal (e.g. `150 == [150]`). Suppresses false positives from storage format differences between profiles. |
| `--diff-changed-only` | Diff mode only: show only changed values. Suppresses the "Only in baseline" and "Only in diff" sections. |
| `--conf FILE` | Conf file for `--auto`. Bare filename is searched within the `-s` directories; any path with separators is used directly. |

#### Output

| Flag | Description |
|------|-------------|
| `--ignore-missing` | Warn and skip inherited files that cannot be found, rather than erroring. |
| `--output FILE` | Write output to FILE instead of stdout (implies `--no-color`). |

---

<a name="output-format"></a>

## 📄 Output Format

### Single Profile Mode

Output is divided into three sections:

#### Header

Shows the full resolved inheritance chain for each section, child-first. When using `--auto`, roles sourced from the conf are annotated with `(*)` and the active bed type is shown:

```
==============================================================================
  OrcaSlicer Effective Settings
==============================================================================
  MACHINE (*)
    My Printer.json  -->
    fdm_vendor_common.json  -->
    fdm_machine_common.json  -->
    fdm_machine_base.json
  FILAMENT (*)
    Generic PLA.json  -->
    fdm_filament_pla_common.json  -->
    fdm_filament_base.json
  PROCESS (*)
    0.2mm Standard.json  -->
    fdm_process_base.json
  (*) Derived from orcaslicer.conf  --auto
  Active Bed Type:  Smooth High Temp Plate  (curr_bed_type=3)
  --------------------------------------------------------------------------
```

Without `--auto` the `(*)` annotation and bed type line are omitted. With `--show-paths`, full file paths are shown instead of filenames.

#### Effective Values

Every setting with its winning value. Settings marked `(*)` were arrived at via one or more overrides. Settings marked `(?)` have a filament override key set but no base key found to compare against:

```
==============================================================================
  Effective Values
==============================================================================
  (* = overridden)  (? = filament override, no base found)
  --------------------------------------------------------------------------

  bed_temperature: 35
  cool_plate_temp: 35
  enable_overhang_bridge_fan: 1  (*)
  fan_max_speed: 100  (*)
  ironing_inset: 2  (?)
  retraction_speed: 45  (*)
  travel_speed: 250  (*)
```

With `--color-code-values`, each key name is colored by its winning section (blue=machine, green=filament, yellow=process) rather than the default white. With `--overrides-only`, only `(*)` and `(?)` lines are shown. With `--values-only`, output stops after this section.

#### Override Detail

Every overridden setting showing each contributing value, lowest priority first. The last (winning) entry is highlighted:

```
==============================================================================
  Override Detail
==============================================================================
  Section overrides only. Run with --show-inheritance to see inherit-chain detail.
  --------------------------------------------------------------------------

  retraction_speed: 45
    [40]  (fdm_machine_common.json: retraction_speed)
    [45]  (Generic PLA.json: filament_retraction_speed)

  travel_speed: 250
    [200]  (fdm_machine_common.json: travel_speed)
    [250]  (My Printer.json: travel_speed)
```

With `--show-inheritance`, every file in the full inheritance chain is shown rather than just the section tip, and the header note changes to reflect this.

#### Counts

```
==============================================================================
  Counts
==============================================================================
  Total settings    : 56
  No override       : 50
  Section overrides : 6
    filament_ prefix  : 1
  Inherit overrides : 10 (suppressed — use --show-inheritance)
  No base found (?) : 1
```

`Total settings` and `Section overrides` reflect the displayed set (narrowed by `--overrides-only` or `--filter` if used). `No override` always reflects the full dataset regardless of display filters, so you can see the full picture at a glance. `Inherit overrides` is only shown if any exist; when suppressed (default), it prompts you to use `--show-inheritance`.

### Diff Mode

Diff mode is triggered whenever any `-d` / `--diff-*` flag is provided. Output shows categorized sections followed by a summary.

```
==============================================================================
  OrcaSlicer Profile Diff
==============================================================================
  BASELINE
    FILAMENT
      Generic PLA.json  -->
      fdm_filament_pla_common.json  -->
      fdm_filament_base.json
  DIFF
    FILAMENT
      Generic PETG.json  -->
      fdm_filament_petg_common.json  -->
      fdm_filament_base.json
==============================================================================

==============================================================================
  Changed  (4)
==============================================================================
  bed_temperature:  35  ->  70
  fan_max_speed:  100  ->  60
  filament_max_volumetric_speed:  21  ->  14
  nozzle_temperature:  220  ->  240

==============================================================================
  Only in baseline  (2)
==============================================================================
  cool_plate_temp: 35
  cool_plate_temp_initial_layer: 35

==============================================================================
  Only in diff  (1)
==============================================================================
  eng_plate_temp: 70

==============================================================================
  Summary
==============================================================================
  Changed          : 4
  Only in baseline : 2
  Only in diff     : 1
  Identical        : 49
```

With `--ignore-type`, values that differ only in scalar-vs-array form (e.g. `150` vs `[150]`) are separated into an additional `Ignored (type mismatch)` section shown in dim text, and counted separately in the Summary. These are suppressed from the `Changed` section to avoid noise when comparing profiles from different OrcaSlicer versions that store the same value in different forms.

With `--diff-changed-only`, the `Only in baseline` and `Only in diff` sections and their Summary lines are suppressed — only `Changed` (and `Ignored` if applicable) are shown.

---

<a name="examples"></a>

## 💡 Examples

### View the active profile (auto mode)

The simplest invocation — reads whatever OrcaSlicer currently has loaded:

*Linux / macOS:*
```bash
python3 orcahunter.py --auto -s ~/.config/OrcaSlicer
```

*Windows:*
```bat
python orcahunter.py --auto -s "%APPDATA%\OrcaSlicer"
```

### View the full effective settings for a printer/filament/process combination

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer/system \
  -s ~/.config/OrcaSlicer/user \
  -b "My Printer 0.4 nozzle.json" \
  -b "Generic PLA.json" \
  -b "0.2mm Standard.json"
```

*Windows:*
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer\system" ^
  -s "%APPDATA%\OrcaSlicer\user" ^
  -b "My Printer 0.4 nozzle.json" ^
  -b "Generic PLA.json" ^
  -b "0.2mm Standard.json"
```

### View only settings that are overridden (no-noise mode)

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer \
  -b "My Printer.json" \
  -b "Generic PLA.json" \
  -b "0.2mm Standard.json" \
  --overrides-only
```

*Windows:*
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer" ^
  -b "My Printer.json" ^
  -b "Generic PLA.json" ^
  -b "0.2mm Standard.json" ^
  --overrides-only
```

### Quick lookup of a specific setting

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer \
  -b "My Printer.json" \
  -b "Generic PLA.json" \
  --filter retraction \
  --values-only
```

*Windows:*
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer" ^
  -b "My Printer.json" ^
  -b "Generic PLA.json" ^
  --filter retraction ^
  --values-only
```

### Show the full inheritance chain detail for overridden settings

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer \
  -b "My Printer.json" \
  -b "Generic PLA.json" \
  -b "0.2mm Standard.json" \
  --show-inheritance --overrides-only
```

*Windows:*
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer" ^
  -b "My Printer.json" ^
  -b "Generic PLA.json" ^
  -b "0.2mm Standard.json" ^
  --show-inheritance --overrides-only
```

### Diff two filament profiles against the same machine and process

Machine and process are shared from baseline — only specify what changes:

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer \
  -b "My Printer.json" \
  -b "Generic PLA.json" \
  -b "0.2mm Standard.json" \
  -d "Generic PETG.json"
```

*Windows:*
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer" ^
  -b "My Printer.json" ^
  -b "Generic PLA.json" ^
  -b "0.2mm Standard.json" ^
  -d "Generic PETG.json"
```

### Diff the active profile against a candidate filament

*Linux / macOS:*
```bash
python3 orcahunter.py \
  --auto \
  -s ~/.config/OrcaSlicer \
  -d "My Custom PLA.json"
```

*Windows:*
```bat
python orcahunter.py ^
  --auto ^
  -s "%APPDATA%\OrcaSlicer" ^
  -d "My Custom PLA.json"
```

### Diff and show only what changed (suppress unique keys)

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer \
  -b "Generic PLA.json" \
  -d "Generic PETG.json" \
  --diff-changed-only
```

*Windows:*
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer" ^
  -b "Generic PLA.json" ^
  -d "Generic PETG.json" ^
  --diff-changed-only
```

### Suppress type-format false positives when diffing across versions

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s /opt/orcaslicer_v1/resources/profiles \
  -b "Generic PLA.json" \
  -S /opt/orcaslicer_v2/resources/profiles \
  -d "Generic PLA.json" \
  --ignore-type
```

*Windows:*
```bat
python orcahunter.py ^
  -s "C:\Program Files\OrcaSlicer_v1\resources\profiles" ^
  -b "Generic PLA.json" ^
  -S "C:\Program Files\OrcaSlicer_v2\resources\profiles" ^
  -d "Generic PLA.json" ^
  --ignore-type
```

### Diff a custom filament against a system filament

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer/system \
  -s ~/.config/OrcaSlicer/user \
  --base-filament "Generic PLA.json" \
  --diff-filament "My Custom PLA.json"
```

*Windows:*
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer\system" ^
  -s "%APPDATA%\OrcaSlicer\user" ^
  --base-filament "Generic PLA.json" ^
  --diff-filament "My Custom PLA.json"
```

### Diff two complete profiles — different machine and filament

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer \
  -b "Printer A.json" \
  -b "Generic PLA.json" \
  -b "0.2mm Standard.json" \
  -d "Printer B.json" \
  -d "Generic PETG.json"
```

*Windows:*
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer" ^
  -b "Printer A.json" ^
  -b "Generic PLA.json" ^
  -b "0.2mm Standard.json" ^
  -d "Printer B.json" ^
  -d "Generic PETG.json"
```

### Save output to a file for archiving or sharing

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer \
  -b "My Printer.json" \
  -b "Generic PLA.json" \
  -b "0.2mm Standard.json" \
  --output my_profile_dump.txt
```

*Windows:*
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer" ^
  -b "My Printer.json" ^
  -b "Generic PLA.json" ^
  -b "0.2mm Standard.json" ^
  --output my_profile_dump.txt
```

### Explicit roles (files not in a standard machine/filament/process directory)

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer \
  --base-machine "My Printer.json" \
  --base-filament "Generic PLA.json" \
  --base-process "0.2mm Standard.json"
```

*Windows:*
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer" ^
  --base-machine "My Printer.json" ^
  --base-filament "Generic PLA.json" ^
  --base-process "0.2mm Standard.json"
```

### Skip inherited files that cannot be found

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s ~/.config/OrcaSlicer \
  -b "Experimental PLA.json" \
  --ignore-missing
```

*Windows:*
```bat
python orcahunter.py ^
  -s "%APPDATA%\OrcaSlicer" ^
  -b "Experimental PLA.json" ^
  --ignore-missing
```

### Use separate search directories for baseline and diff

Useful when comparing profiles from two different OrcaSlicer installations or versions:

*Linux / macOS:*
```bash
python3 orcahunter.py \
  -s /opt/orcaslicer_v1/resources/profiles \
  -b "My Printer.json" \
  -b "Generic PLA.json" \
  -S /opt/orcaslicer_v2/resources/profiles \
  -d "Generic PLA.json"
```

*Windows:*
```bat
python orcahunter.py ^
  -s "C:\Program Files\OrcaSlicer_v1\resources\profiles" ^
  -b "My Printer.json" ^
  -b "Generic PLA.json" ^
  -S "C:\Program Files\OrcaSlicer_v2\resources\profiles" ^
  -d "Generic PLA.json"
```

---

<a name="color-customization"></a>

## 🎨 Color Customization

Output colors are configured in the `colors` block of `orcahunter_config.json` — no code editing required:

```json
"colors": {
    "key":          "bright_white",
    "value":        "cyan",
    "marker":       "bright_yellow",
    "machine":      "blue",
    "filament":     "green",
    "process":      "yellow",
    "source":       "dim",
    "header":       "bold",
    "winner":       "bright_cyan",
    "loser":        "red",
    "diff_changed": "bright_yellow"
}
```

| Role           | Controls                                                      |
|----------------|---------------------------------------------------------------|
| `key`          | Setting key names (also used as the base by `--color-code-values`; overridden per-section when active) |
| `value`        | Effective values                                              |
| `marker`       | `(*)` override marker                                         |
| `machine`      | Machine section labels, sources, and key names (with `--color-code-values`) |
| `filament`     | Filament section labels, sources, and key names (with `--color-code-values`) |
| `process`      | Process section labels, sources, and key names (with `--color-code-values`) |
| `source`       | Source filenames in override detail and dim annotations       |
| `header`       | Section headers                                               |
| `winner`       | Winning (effective) value in override detail                  |
| `loser`        | Losing (overridden) values in override detail                 |
| `diff_changed` | Changed value on the diff side in diff mode                   |

Valid color names: `black`, `red`, `green`, `yellow`, `blue`, `magenta`, `cyan`, `white`, and their `bright_` variants (e.g. `bright_cyan`), plus `bold` and `dim`. An invalid color name will produce an error at startup identifying the bad entry.

Color output is automatically disabled when writing to a file (`--output`) or when stdout is not a terminal. Use `--no-color` to disable it explicitly.

---

<a name="extending-the-config"></a>

## 🔧 Extending the Config

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

---

<a name="limitations"></a>

## ⚠️ Limitations

### Settings with no inherited baseline

`orcahunter` determines whether a setting is overridden by comparing values across the resolved inheritance chain. A `filament_` override key can only be confirmed as an override if the corresponding base key also appears somewhere in the chain to compare against.

If you set a filament override key in your profile — for example `filament_ironing_inset` — but none of the ancestor files define the corresponding base key (`ironing_inset`), `orcahunter` has nothing to compare against. The base value OrcaSlicer uses in this case is a compiled-in internal default that is never written to any profile JSON file. Since `orcahunter` only reads JSON files, it cannot know what that default is.

Rather than silently presenting the value as unoverridden, `orcahunter` marks it with `(?)` to indicate that a filament override key is present but no base was found. The value is real and will be used by OrcaSlicer — the `(?)` signals that the override relationship cannot be fully verified from the files alone. These settings appear in their own subsection of the Override Detail output and are counted separately under `No base found (?)` in the Counts.

**Example:** Setting `filament_ironing_inset` in a filament profile when no ancestor defines `ironing_inset`:

```
  ironing_flow: 16  (*)     ← confirmed override, ancestor had a different value
  ironing_inset: 2  (?)     ← filament override key set, but no base key found to compare
  ironing_spacing: 0.2  (*) ← confirmed override
  ironing_speed: 14  (*)    ← confirmed override
```

The most common cause is a setting that was added to OrcaSlicer after the vendor base profiles were last updated, or a setting that the vendor simply never explicitly defined (relying on OrcaSlicer's compiled default instead).
