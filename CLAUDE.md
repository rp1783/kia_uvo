# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Home Assistant custom integration for Kia Uvo and Hyundai Bluelink vehicle connectivity services. It provides cloud polling of vehicle data and remote control capabilities for Kia, Hyundai, and Genesis vehicles across multiple regions (EU, US, Canada, Australia, New Zealand, India, China, Brazil).

The integration relies on the external Python package `hyundai_kia_connect_api` (version 3.51.5) to communicate with vehicle APIs.

## Development Environment

### Using DevContainer (Recommended)
The project includes a devcontainer setup for consistent development environments:
```bash
# DevContainer will automatically run: container install
# This sets up a Home Assistant development environment with the integration loaded
# Access Home Assistant at http://localhost:8123
```

### Manual Setup with Pre-commit
```bash
# Install pre-commit hooks
pre-commit install

# Run all checks manually
pre-commit run --all-files

# Run specific hooks
pre-commit run ruff --all-files
pre-commit run mypy --all-files
```

## Code Quality Commands

### Linting and Formatting
```bash
# Format code with ruff
ruff format custom_components/kia_uvo/

# Lint and auto-fix with ruff
ruff check --fix custom_components/kia_uvo/

# Type checking with mypy (strict mode)
mypy --strict --ignore-missing-imports custom_components/kia_uvo/
```

### Validation
```bash
# These are run in CI but can't be easily run locally without Home Assistant tooling
# They validate the integration manifest and HACS compatibility
# - hassfest (Home Assistant manifest validator)
# - HACS action (validates HACS compatibility)
```

### Pre-commit Hooks
The following checks run automatically on commit:
- **ruff**: Python linting and auto-fixing
- **ruff-format**: Code formatting
- **codespell**: Spell checking (ignores: fro, hass, fatc)
- **pyupgrade**: Python syntax modernization
- **mypy**: Type checking with strict mode
- **prettier**: JSON/YAML/Markdown formatting
- **yamllint**: YAML validation
- **trailing-whitespace**: Removes trailing whitespace
- **end-of-file-fixer**: Ensures files end with newline

Manual hooks (run with `pre-commit run --hook-stage manual <hook-name> --all-files`):
- **python-typing-update**: Updates Python typing syntax to 3.11+ style

## Architecture

### Core Components

**Coordinator** (`coordinator.py`):
- `HyundaiKiaConnectDataUpdateCoordinator` extends Home Assistant's `DataUpdateCoordinator`
- Manages authentication token refresh via `VehicleManager` from `hyundai_kia_connect_api`
- Handles two update modes:
  - **Cached updates**: Fetches server-cached vehicle data (default every 30 minutes, configurable)
  - **Force updates**: Requests fresh data directly from vehicle (default every 4 hours/1440 minutes, configurable)
- Implements blackout hours to prevent force updates during sleep hours (default 10PM-7AM, configurable)
- All vehicle actions (lock, climate, charge, etc.) are async and create background tasks that wait for action completion before refreshing

**Entity Base** (`entity.py`):
- `HyundaiKiaConnectEntity` extends `CoordinatorEntity`
- Provides common device info structure for all entities
- Links entities to vehicles via `vehicle.id` as the device identifier

**Config Flow** (`config_flow.py`):
- Multi-step configuration: region/brand selection, then credentials
- Options flow for configuring update intervals and geolocation settings
- Handles config migration from version 1 to version 2
- Supports EU special authentication (token-based for Kia Europe)

**Services** (`services.py`):
- Exposes 17 vehicle control services (update, force_update, lock, unlock, start_climate, stop_climate, etc.)
- All services use `device_id` to identify the target vehicle
- Services handle single-vehicle/single-account shortcuts automatically
- Complex services like `start_climate` and `schedule_charging_and_climate` accept many optional parameters

### Platform Files
- `binary_sensor.py`: Door/window/lock/charge status sensors
- `sensor.py`: Numeric and state sensors (battery, range, odometer, etc.)
- `device_tracker.py`: GPS location tracking
- `lock.py`: Lock/unlock controls
- `number.py`: Numeric controls (charge limits, V2L settings, etc.)
- `climate.py`: Currently commented out in PLATFORMS list

### Constants (`const.py`)
- Domain: `kia_uvo`
- Supported regions: Europe, Canada, USA, China, Australia, India, New Zealand, Brazil
- Supported brands: Kia (1), Hyundai (2), Genesis (3)
- Default intervals: 30 min cache update, 1440 min (4 hour) force update
- Default blackout: 22:00 (10PM) to 07:00 (7AM)

### Data Flow
1. User configures integration with credentials and region/brand
2. `async_setup_entry` creates a `HyundaiKiaConnectDataUpdateCoordinator` instance
3. Coordinator initializes `VehicleManager` from `hyundai_kia_connect_api` package
4. Periodic updates fetch vehicle data via coordinator's `_async_update_data`
5. Entities observe coordinator data changes and update their states
6. Services call coordinator methods which invoke `VehicleManager` actions and create background refresh tasks

### Authentication
- EU Kia/Hyundai: Token-based (requires special login flow documented in upstream API)
- Other regions: Username/password with optional PIN
- Token refresh is automatic via `check_and_refresh_token()`

## Important Integration Details

### Update Behavior
- The coordinator calculates update interval as `min(scan_interval, force_refresh_interval)`
- Force updates only occur outside blackout hours
- If force update fails, falls back to cached update
- After any vehicle action (lock, climate, etc.), a 5-second delay occurs before checking action status and refreshing

### Service Device Targeting
- If only one account and one vehicle exist, device_id is optional
- Otherwise, device_id is required to identify the target vehicle
- Device identifiers use the format `(DOMAIN, vehicle.id)`

### Multi-Account and Multi-Vehicle Support
- Each config entry represents one account
- One account can have multiple vehicles
- Multiple accounts can be configured (run setup multiple times)
- Each coordinator is stored with its `config_entry.unique_id` as the key

### Version Migration
- Config version 1 to 2 migration occurs in `async_migrate_entry`
- Migration updates unique_id to SHA256 hash of brand/region/username
- All entities are removed and recreated during migration

## Development Notes

### Python Version
- Target: Python 3.11+ (per pyupgrade config: `--py311-plus`)
- Type hints use modern syntax (enforced by `python-typing-update` hook)

### Dependency Management
- The integration version in `manifest.json` must be updated manually
- The `hyundai_kia_connect_api` version is pinned in `manifest.json` requirements
- Home Assistant minimum version: 2025.1

### Translation Files
- Located in `custom_components/kia_uvo/translations/`
- Base strings in `strings.json`
- Service definitions in `services.yaml` (used for Home Assistant service UI)

### Testing
- No test suite currently exists in this repository
- Manual testing via devcontainer is the primary validation method
- CI runs hassfest (HA manifest validator) and HACS validation only

### Code Style
- Use ruff for formatting (replaces black)
- Strict mypy type checking required
- Max line length and other style rules enforced by ruff
- f-strings preferred for string formatting

## Common Patterns

### Adding a New Service
1. Define service constant in `services.py` (e.g., `SERVICE_NEW_FEATURE`)
2. Add to `SUPPORTED_SERVICES` tuple
3. Create async handler function `async_handle_new_feature(call)`
4. Add method to coordinator if it requires API calls
5. Register in `services` dict in `async_setup_services`
6. Define service schema in `services.yaml`
7. Add translations to `strings.json`

### Adding a New Entity
1. Create entity class extending `HyundaiKiaConnectEntity`
2. Override properties: `name`, `unique_id`, `state`, `icon`, etc.
3. Add platform to `PLATFORMS` list in `__init__.py`
4. Create platform setup function `async_setup_entry(hass, config_entry, async_add_entities)`
5. Check vehicle capabilities before creating entities (not all vehicles support all features)

### Accessing Vehicle Data
```python
# In entity classes
self.vehicle  # Vehicle object from hyundai_kia_connect_api
self.coordinator.data  # Latest coordinator data
self.coordinator.vehicle_manager  # Access to VehicleManager

# Common vehicle properties (from hyundai_kia_connect_api)
self.vehicle.id
self.vehicle.name
self.vehicle.model
self.vehicle.VIN
# ... and many vehicle state properties
```

## Important Constraints

- Do not commit secrets or credentials
- Respect blackout hours for force updates (user sleep time)
- Always handle authentication errors by raising `ConfigEntryAuthFailed`
- Services must validate required parameters and log errors for missing values
- Region/brand compatibility varies - check upstream API documentation for feature support
