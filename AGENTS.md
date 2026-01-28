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
    ├── description.md       # Short app description (required)
    └── logo.jpg            # App logo image (required)
```

### Key Files
- `apps/app-info-schema.json`: JSON schema defining the structure of config.json files
- `apps/dynamic-compose-schema.json`: JSON schema defining the structure of docker-compose.json files
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
- `tipi_version`: Integer version for the Runtipi platform (always 1 for new apps, increment when updating)
- `short_desc`: Brief description
- `author`: App author/maintainer (GitHub username)
- `source`: Source code repository URL
- `categories`: Array from predefined list (network, media, development, automation, social, utilities, photography, security, featured, books, data, music, finance, gaming, ai)
- `description`: Long description of the app
- `version`: The actual version of the app (not the Runtipi version)
- `exposable`: Boolean for whether app can be exposed externally
- `supported_architectures`: Array of "amd64" and/or "arm64"

Common optional fields:
- `port`: Main port number (1-65535)
- `website`: Link to the official website
- `force_expose`: If true, the app will require a domain name
- `generate_vapid_keys`: If true, generates VAPID keys for web push (VAPID_PUBLIC_KEY and VAPID_PRIVATE_KEY available as env vars)
- `url_suffix`: If set, the app will be accessible at https://<your-domain>/<url_suffix>
- `https`: If true, the app will be accessible via https only
- `no_gui`: Set to true for apps with no GUI (Open button will be hidden)
- `uid`, `gid`: Specific permissions for the app's data folder (Runtipi will chown the data directory; both must be specified)
- `min_tipi_version`: Minimum Runtipi version required (e.g., "v3.0.0")
- `form_fields`: Array of configuration form fields for user input
- `dynamic_config`: Boolean indicating if app uses docker-compose.json (use `dynamic` instead in newer versions)
- `dynamic`: Use the new docker-compose.json to dynamically generate a compose file
- `deprecated`: If true, app won't exist in the app store and will notify users it's no longer maintained
- `created_at`: Unix timestamp (milliseconds) when the app was created
- `updated_at`: Unix timestamp (milliseconds) when the app was last updated

### Form Fields
Apps can request user input during installation using `form_fields` in config.json. Each field has:
- `type` (required): Field type - text, password, email, number, fqdn, ip, fqdnip, random, boolean
- `label` (required): Label shown to user
- `env_variable` (required): Environment variable name for use in docker-compose files
- `required` (required): Whether the field is required
- `hint`: Small hint to show the user
- `placeholder`: Placeholder text for the field
- `default`: Default value (used only if required is false)
- `regex`: Regex pattern to verify user input
- `pattern_error`: Error message when regex validation fails
- `min`: Minimum length for text/password input or value for number
- `max`: Maximum length for text/password input or value for number
- `options`: Array of {label, value} objects for dropdown selection
- `encoding`: For random type only - specify "base64" or "hex" encoding

Notes on form fields:
- For `random` type, `min` determines the generated string length (default: 32 characters)
- Use `options` only for pre-defined values; use text field with examples in label for custom values
- The `random` type generates values like: `2e318419a49b70ad93766a5d6eb54d9ebbcceaeadd57c5f6897dbbe10afbc880`

Example:
```json
{
  "form_fields": [
    {
      "type": "text",
      "label": "Username",
      "max": 50,
      "min": 3,
      "required": true,
      "env_variable": "NEXTCLOUD_ADMIN_USER"
    },
    {
      "type": "password",
      "label": "Password",
      "max": 50,
      "min": 3,
      "required": true,
      "env_variable": "NEXTCLOUD_ADMIN_PASSWORD"
    },
    {
      "type": "text",
      "label": "Select your favorite fruits",
      "required": true,
      "options": [
        {"label": "Apple", "value": "apple"},
        {"label": "Banana", "value": "banana"}
      ],
      "env_variable": "FRUITS"
    }
  ]
}
```

### docker-compose.json Structure
The dynamic compose format uses a simplified service definition that allows more control over app deployment.

#### Top-Level Configuration
- `schemaVersion`: Must be set to `2` (required)
- `services`: Array of service objects (required)
- `overrides`: Array of architecture-specific overrides (optional)

#### Basic Service Configuration
Each service in the `services` array requires:
- `name` (required): The name of the service and container
- `image` (required): The Docker image to use (e.g., "nginx:latest")
- `command`: String or array - command to run in the container (e.g., "/my/app" or ["npm", "start"])
- `environment`: Array of environment variables with `key` and `value` fields (NOT object format)
  ```json
  "environment": [
    {"key": "FOO", "value": "bar"},
    {"key": "PORT", "value": 8080},
    {"key": "ENABLE_FEATURE", "value": true}
  ]
  ```

#### Port Configuration
- `internalPort`: The main port exposed by the service (number)
- `addPorts`: Array of additional ports to expose (each has `containerPort`, `hostPort`, `tcp`/`udp`)

#### Resource Configuration
- `volumes`: Array of volume mounts (each has `hostPath` and `containerPath`)
- `mounts`: Array of bind mounts (similar to volumes)

#### Security Configuration
- `capAdd`: Array of Linux capabilities to add (e.g., ["NET_ADMIN"])
- `capDrop`: Array of Linux capabilities to drop
- `privileged`: Boolean - run container in privileged mode
- `networkMode`: String - network mode (e.g., "host", "bridge")

#### Advanced Configuration
- `isMain`: Boolean marking the primary service (exactly one service per app must have this set to true)
- `dependsOn`: Array of service names this service depends on
- `healthCheck`: Object with health check configuration
- `extraHosts`: Array of additional host mappings
- `labels`: Object with container labels
- `user`: String - user to run the container as
- `workingDir`: String - working directory in the container
- `entrypoint`: String or array - override the default entrypoint

#### Architecture Overrides
You can provide architecture-specific configurations in the `overrides` array:
```json
{
  "overrides": [
    {
      "architecture": "arm64",
      "services": [
        {
          "name": "myapp",
          "image": "myapp:arm64"
        }
      ]
    }
  ]
}
```

Supported architectures: "amd64", "arm64"

#### Complete Example
```json
{
  "schemaVersion": 2,
  "services": [
    {
      "name": "nginx",
      "image": "nginx:latest",
      "isMain": true,
      "internalPort": 80,
      "environment": [
        {"key": "NGINX_HOST", "value": "example.com"},
        {"key": "NGINX_PORT", "value": 80}
      ],
      "volumes": [
        {"hostPath": "/app/data", "containerPath": "/usr/share/nginx/html"}
      ]
    }
  ]
}
```

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
Avoid using `latest` tags in Docker images as they can break your apps in the future.

### Adding a New App
1. Create directory: `apps/<app-id>/`
2. Create required files:
   - `config.json` with all required fields
   - `docker-compose.json` OR `docker-compose.yml`
   - `metadata/description.md`
   - `metadata/logo.jpg`
3. Ensure `$schema` fields point to schemas in apps/ directory:
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
