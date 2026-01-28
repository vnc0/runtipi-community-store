# AGENTS.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Overview

This is a Runtipi community app store template repository. It contains a collection of containerized applications that can be deployed through the Runtipi platform. Each app is defined by configuration files, Docker compose definitions, and metadata.

## Official Documentation

Refer to these official Runtipi docs for detailed information:
- [Create Your Own App Store Guide](https://runtipi.io/docs/guides/create-your-own-app-store)
- [Dynamic Compose File Guide](https://runtipi.io/docs/guides/dynamic-compose-guide)
- [config.json Options Reference](https://runtipi.io/docs/reference/config-json)

## Development Commands

### Testing
Run the full test suite to validate all apps:
```bash
bun test
```

Run tests for a specific app by filtering:
```bash
bun test --test-name-pattern "app <app-name>"
```

### Installing Dependencies
```bash
bun install
```

### Updating App Configuration
Update an app's config.json after version changes (increments tipi_version, updates version and timestamp):
```bash
bun ./scripts/update-config.ts apps/<app-name>/docker-compose.json <new-version>
```

## Repository Structure

### App Directory Layout
Each app in `apps/` follows this structure:
```
apps/<app-id>/
├── config.json              # App metadata and configuration (validated against app-info-schema.json)
├── docker-compose.json      # Dynamic compose config (validated against dynamic-compose-schema.json)
├── docker-compose.yml       # Traditional Docker Compose file (optional, for reference)
└── metadata/
    ├── description.md       # App description (required)
    └── logo.jpg            # App logo image (required)
```

### Key Files
- `app-info-schema.json`: JSON schema defining the structure of config.json files
- `dynamic-compose-schema.json`: JSON schema defining the structure of docker-compose.json files
- `docker-compose.common.yml`: Shared network configuration for all apps
- `__tests__/apps.test.ts`: Test suite that validates all apps
- `scripts/update-config.ts`: Utility to update app configurations
- `renovate.json`: Configuration for automated dependency updates via Renovate bot

## App Configuration Guidelines

### config.json Requirements
Every app MUST have these required fields:
- `id`: Unique app identifier (matches directory name)
- `available`: Boolean indicating if app is available
- `name`: Display name
- `tipi_version`: Integer version for the Runtipi platform
- `short_desc`: Brief description
- `author`: App author/maintainer
- `source`: Source code repository URL

Common optional fields:
- `port`: Main port number (1-65535)
- `version`: App version (defaults to "latest")
- `categories`: Array from predefined list (network, media, development, automation, social, utilities, photography, security, featured, books, data, music, finance, gaming, ai)
- `exposable`: Boolean for whether app can be exposed externally
- `dynamic_config`: Boolean indicating if app uses docker-compose.json
- `supported_architectures`: Array of "amd64" and/or "arm64"
- `min_tipi_version`: Minimum Runtipi version required
- `form_fields`: Array of configuration form fields for user input

### docker-compose.json Structure
The dynamic compose format uses a simplified service definition (Schema Version 2):
- `schemaVersion`: Must be set to `2` (required)
- `services`: Array of service objects
  - `name`: Service name (required)
  - `image`: Docker image with tag (required)
  - `internalPort`: Port the service listens on
  - `isMain`: Boolean marking the primary service (one service per app must have this set to true)
  - Additional fields: `volumes`, `environment`, `command`, `networkMode`, `capAdd`, `privileged`, `dependsOn`, `healthCheck`, etc.

Note: Schema v2 uses an array format for environment variables with `key` and `value` fields, not the v1 object format.

### Schema Validation
Both config.json and docker-compose.json are validated against their respective schemas using the `@runtipi/common` package. The test suite will catch validation errors.

## Architecture Patterns

### Two Compose Formats
Apps can use either:
1. **Dynamic compose** (docker-compose.json): Simplified JSON format that Runtipi transforms
2. **Standard compose** (docker-compose.yml): Traditional Docker Compose YAML (may be kept for reference)

Most apps in this store use the dynamic_config approach with docker-compose.json.

### Networking
All apps connect to the `tipi_main_network` (external network named `runtipi_tipi_main_network`). This is defined in docker-compose.common.yml and referenced in individual app compose files.

### Version Management
- `version` field in config.json tracks the application version
- `tipi_version` is an integer that increments with each configuration change
- `updated_at` is a Unix timestamp (milliseconds) of the last update
- These are automatically updated by the update-config.ts script during renovate updates

### Automated Updates
Renovate bot:
- Monitors Docker images in docker-compose.json files
- Creates PRs for version updates
- Automatically runs the update-config.ts script to sync config.json
- Runs tests to validate changes
- Does NOT auto-update database images (mariadb, mysql, mongodb, postgres, redis)

## Working with Apps

### Adding a New App
1. Create directory: `apps/<app-id>/`
2. Create required files:
   - `config.json` with all required fields
   - `docker-compose.json` OR `docker-compose.yml`
   - `metadata/description.md`
   - `metadata/logo.jpg`
3. Ensure `$schema` fields point to parent schemas:
   - In config.json: `"$schema": "../app-info-schema.json"`
   - In docker-compose.json: `"$schema": "../dynamic-compose-schema.json"`
4. Run `bun test` to validate

### Modifying an App
When updating an existing app:
1. Modify the necessary files (config.json, docker-compose.json, etc.)
2. If changing the Docker image version, use the update-config script or manually increment `tipi_version` and update `updated_at` timestamp in config.json
3. Run `bun test` to ensure validation passes

### Testing Individual Apps
The test suite validates each app for:
- Presence of all required files
- Valid config.json against the schema
- Valid docker-compose.json against the schema

To debug validation errors, check the test output which includes detailed zod validation error messages.
