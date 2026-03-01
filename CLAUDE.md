# Home Assistant Configuration Management

This repository manages Home Assistant configuration files with automated validation, testing, and deployment.

## Before Making Changes

**Always consult the latest Home Assistant documentation** at https://www.home-assistant.io/docs/ before suggesting configurations, automations, or integrations. HA updates frequently and syntax/features change between versions.

## Project Structure

- `config/` - Contains all Home Assistant configuration files (synced from HA instance)
- `tools/` - Validation and testing scripts
- `venv/` - Python virtual environment with dependencies
- `temp/` - Temporary directory for Claude to write and test code before moving to final locations
- `Makefile` - Commands for pulling/pushing configuration
- `.claude-code/` - Project-specific Claude Code settings and hooks
  - `hooks/` - Validation hooks that run automatically
  - `settings.json` - Project configuration

## Rsync Architecture

This project uses **two separate exclude files** for different sync operations:

| File | Used By | Purpose |
|------|---------|---------|
| `.rsync-excludes-pull` | `make pull` | Less restrictive |
| `.rsync-excludes-push` | `make push` | More restrictive |

**Why separate files?**
- `make pull` downloads most files including `.storage/` (excluding sensitive auth files) for local reference
- `make push` writes never overwrites HA's runtime state (`.storage/`)

## What This Repo Can and Cannot Manage

### SAFE TO MANAGE (YAML files)
- `automations.yaml` - Automation definitions
- `scenes.yaml` - Scene definitions
- `scripts.yaml` - Script definitions
- `configuration.yaml` - Main configuration
- `secrets.yaml` - Secret values

### NEVER MODIFY LOCALLY (Runtime State)
These files in `.storage/` are managed by Home Assistant at runtime. Local modifications will be **overwritten** by HA on restart or ignored entirely.

### Entity/Device Changes (Manual Only)

Do not change entities or devices programmatically from this repo. If changes are
needed, make them manually in the Home Assistant UI:
- Settings → Devices & Services → Entities → Edit

### Reloading After YAML Changes
- Automations: `POST /api/services/automation/reload`
- Scenes: `POST /api/services/scene/reload`
- Scripts: `POST /api/services/script/reload`

## Workflow Rules

### Before Making Changes
1. Run `make pull` to ensure local files are current
2. Identify if the change affects YAML files or `.storage/` files
3. YAML files → edit locally, then `make push`
4. `.storage/` files → use the HA UI only (manual changes)

### Before Running `make push`
1. Validation runs automatically - do not push if validation fails
2. Only YAML configuration files will be synced (`.storage/` is protected)

### After `make push`
1. Reload the relevant HA components (automations, scenes, scripts)
2. Verify changes took effect in HA

## Available Commands

### Configuration Management
- `make pull` - Pull latest config from Home Assistant instance
- `make push` - Push local config to Home Assistant (with validation)
- `make backup` - Create backup of current config
- `make validate` - Run all validation tests

### Validation Tools
- `python tools/run_tests.py` - Run complete validation suite
- `python tools/yaml_validator.py` - YAML syntax validation only
- `python tools/reference_validator.py` - Entity/device reference validation
- `python tools/ha_official_validator.py` - Official HA configuration validation

### Entity Discovery Tools
- `make entities` - Explore available Home Assistant entities
- `python tools/entity_explorer.py` - Entity registry parser and explorer
  - `--search TERM` - Search entities by name, ID, or device class
  - `--domain DOMAIN` - Show entities from specific domain (e.g., climate, sensor)
  - `--area AREA` - Show entities from specific area
  - `--full` - Show complete detailed output

## Validation System

This project includes comprehensive validation to prevent invalid configurations:

### Core Goal

- Verify all agent-produced configuration and automation changes before saving YAML files to Home Assistant
- Never generate, save, or push YAML changes that fail validation
- Use a layered validation suite that combines Home Assistant's own validation with repository-specific validators

1. **YAML Syntax Validation** - Ensures proper YAML syntax with HA-specific tags
2. **Entity Reference Validation** - Checks that all referenced entities/devices exist
3. **Official HA Validation** - Uses Home Assistant's own validation tools

### Automated Validation Hooks

- **Post-Edit Hook**: Runs validation after editing any YAML files in `config/`
- **Pre-Push Hook**: Validates configuration before pushing to Home Assistant
- **Blocks invalid pushes**: Prevents uploading broken configurations

## Home Assistant Instance Details

- **Host**: Configure in Makefile `HA_HOST` variable
- **User**: Configure SSH access as needed
- **SSH Key**: Configure SSH key authentication
- **Config Path**: /config/ (standard HA path)
- **Version**: Compatible with Home Assistant Core 2024.x+

## Entity Registry

The system tracks entities across these domains:
- alarm_control_panel, binary_sensor, button, camera, climate
- device_tracker, event, image, light, lock, media_player
- number, person, scene, select, sensor, siren, switch
- time, tts, update, vacuum, water_heater, weather, zone

## Development Workflow

1. **Pull Latest**: `make pull` to sync from HA
2. **Edit Locally**: Modify files in `config/` directory
3. **Auto-Validation**: Hooks automatically validate on edits
4. **Test Changes**: `make validate` for full test suite
5. **Deploy**: `make push` to upload (blocked if validation fails)

## Key Features

- ✅ **Safe Deployments**: Pre-push validation prevents broken configs
- ✅ **Entity Validation**: Ensures all references point to real entities
- ✅ **Entity Discovery**: Advanced tools to explore and search available entities
- ✅ **Official HA Tools**: Uses Home Assistant's own validation
- ✅ **YAML Support**: Handles HA-specific tags (!include, !secret, !input)
- ✅ **Comprehensive Testing**: Multiple validation layers
- ✅ **Automated Hooks**: Validation runs automatically on file changes

## Important Notes

- **Never push without validation**: The hooks prevent this, but be aware
- **Blueprint files** use `!input` tags which are normal and expected
- **Secrets are skipped** during validation for security
- **SSH access required** for pull/push operations
- **Python venv required** for validation tools
- All python tools need to be run with `source venv/bin/activate && python <tool_path>`

## Troubleshooting

### Validation Fails
1. Check YAML syntax errors first
2. Verify entity references exist in `.storage/` files
3. Run individual validators to isolate issues
4. Check HA logs if official validation fails

### SSH Issues
1. Verify SSH key permissions: `chmod 600 ~/.ssh/your_key`
2. Test connection: `ssh your_homeassistant_host`
3. Check SSH config in `~/.ssh/config`

### Missing Dependencies
1. Activate venv: `source venv/bin/activate`
2. Install requirements: `pip install homeassistant voluptuous pyyaml`

## Security

- **SSH keys** are used for secure access
- **Secrets.yaml** is excluded from validation (contains sensitive data)
- **No credentials** are stored in this repository
- **Access tokens** in config are for authorized integrations

This system ensures you can confidently manage Home Assistant configurations with Claude while maintaining safety and reliability.

## Entity Naming Convention

This Home Assistant setup uses a **standardized entity naming convention** for multi-location deployments:

### **Format: `location_room_device_sensor`**

**Structure:**
- **location**: `home`, `office`, `cabin`, etc.
- **room**: `basement`, `kitchen`, `living_room`, `main_bedroom`, `guest_bedroom`, `driveway`, etc.
- **device**: `motion`, `heatpump`, `sonos`, `lock`, `vacuum`, `water_heater`, `alarm`, etc.
- **sensor**: `battery`, `tamper`, `status`, `temperature`, `humidity`, `door`, `running`, etc.

### **Examples:**
```
binary_sensor.home_basement_motion_battery
binary_sensor.home_basement_motion_tamper
media_player.home_kitchen_sonos
media_player.office_main_bedroom_sonos
climate.home_living_room_heatpump
climate.office_living_room_thermostat
lock.home_front_door_august
sensor.office_driveway_camera_battery
vacuum.home_roborock
vacuum.office_roborock
```

### **Benefits:**
- **Clear location identification** - no ambiguity between properties
- **Consistent structure** - easy to predict entity names
- **Automation-friendly** - simple to target location-specific devices
- **Scalable** - supports additional locations or rooms

### **Implementation:**
- All location-based entities follow this convention
- Legacy entities have been systematically renamed
- New entities should follow this pattern
- Vendor prefixes (aquanta_, august_, etc.) are replaced with descriptive device names

### **Claude Code Integration:**
- When creating automations, always ask the user for input if there are multiple choices for sensors or devices
- Use the entity explorer tools to discover available entities before writing automations
- Follow the naming convention when suggesting entity names in automations

---

## Hard-Won Learnings (2026.x)

### HomeKit Bridge

**Do not define `homekit:` in `configuration.yaml`.** YAML-imported bridges (`source: import`) have a persistent pairing failure: HA computes the mDNS setup hash using an internally generated `setup_id` that doesn't align with the stored pincode, causing iOS to reject every code attempt with "The setup code is incorrect." This cannot be fixed by restarting or clearing state files.

**The only reliable setup path:**
1. Keep `homekit:` out of `configuration.yaml` entirely.
2. HA's `default_config:` auto-creates a "HASS Bridge" entry (`source: user`) on first start.
3. That entry has no pincode in `.storage/core.config_entries` — inject one directly:
   ```python
   # ssh homeassistant
   import json
   with open('/config/.storage/core.config_entries', 'r') as f: data = json.load(f)
   for e in data['data']['entries']:
       if e['domain'] == 'homekit' and 'HASS Bridge' in e['title']:
           e['data']['pincode'] = '234-56-819'  # choose any valid XXX-XX-XXX code
   with open('/config/.storage/core.config_entries', 'w') as f: json.dump(data, f)
   ```
4. Delete the bridge's `.state`, `.aids`, `.iids` files from `.storage/`.
5. Restart HA. Pair on iPhone using the injected code.

**Entity filter** lives in the config entry `options.filter` (not YAML). Edit it directly in `.storage/core.config_entries` and restart HA to apply changes.

**TV entities (`device_class: tv`) must NOT be in the bridge.** The bridge sets `exclude_accessory_mode: True` in `data` specifically to block TV entities. Even if added via `include_entities`, webostv's `assumed_state: True` (when the WebSocket session is not live) causes HomeKit to show "No Response" and commands don't reach the TV. Remove the TV from the bridge and control it via the HA dashboard instead.

**Siri works for:** light switch, Nest Hub / media player (Cast), and future switches/lights added to the setup.

**Diagnosing HomeKit issues via mDNS:**
```bash
dns-sd -B _hap._tcp local.                    # list all HAP accessories
dns-sd -L "HASS Bridge C0726E" _hap._tcp local.  # inspect TXT record (sf=1 = unpaired)
nc -zv homeassistant.local 21064              # verify port is reachable from this machine
```
State files are at `/config/.storage/homekit.<entry_id>.[state|aids|iids]`.

---

### LG WebOS TV (`webostv` integration)

- HA discovers the LG C3 via **Google Cast** first (`media_player.lg_webos_tv_oled65c3pua`, platform: cast). This gives very limited control. Add the native **LG webOS Smart TV** integration separately for full control.
- After adding webostv, the entity is `media_player.tv` (platform: webostv). Update all dashboard/automation references from the Cast entity to this one.
- **TURN_ON is not supported** out of the box. The integration requires a MAC address in `entry.data["mac"]` to send a WoL packet. Get the MAC while the TV is on: `ssh homeassistant "arp -n <tv-ip>"`. Inject into the webostv config entry. Note: if the MAC has the locally-administered bit set (second hex digit is 2, 6, A, or E), it may be randomized and WoL will not work.
- The TV's webOS WebSocket session drops frequently, leaving `assumed_state: True`. This is incompatible with HomeKit. Control the TV via HA UI/app instead of HomeKit/Siri.
- Dashboard should reference `media_player.tv`, not `media_player.lg_webos_tv_oled65c3pua`.

---

### `.storage/` Direct Editing

Several integration settings are only manageable by directly editing `.storage/core.config_entries` (not via YAML or UI). Pattern:
1. SSH in, edit the JSON file with Python.
2. Delete relevant `.storage/homekit.<entry_id>.*` state files if changing HomeKit identity.
3. Restart HA — **do not just reload**, a full restart is required for config entry data changes.

Always use a full HA restart (not component reload) when:
- Adding/removing `homekit:` from `configuration.yaml`
- Editing `.storage/core.config_entries` directly
- Creating new `input_boolean` helpers in YAML (reload only updates existing ones)

---

### HA SSH / Logs

- The SSH session (`ssh homeassistant`) connects to the **Advanced SSH addon container**, not the HA Core container. Direct `ps`, `journalctl`, and log file access are limited.
- HA logs are NOT at `/config/home-assistant.log` in 2026.x. Use the HA UI log viewer or the `/api/error_log` REST endpoint (returns 404 in 2026.x — use the UI instead).
- The HA REST API token is in `.env` as `HA_TOKEN`. Always use it for API calls rather than hardcoding.
- `POST /api/services/homeassistant/restart` triggers a full HA restart. Allow ~60 seconds before checking if HA is back up.
