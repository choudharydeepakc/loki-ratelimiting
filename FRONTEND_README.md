# Loki Frontend Authorization

## Overview
The Loki frontend supports TLS certificate-based authorization to control access to tenant data. This feature can be enabled or disabled via configuration.

## Configuration Options

### Command Line
```bash
./cmd/loki/loki -target=query-frontend -config.file=loki-frontend-config.yaml -frontend.authz-enabled=true -frontend.authz-config-path=/path/to/config.json
```

### YAML Configuration
```yaml
frontend:
  authz_enabled: true                  # Enable TLS certificate-based authorization
  authz_config_path: /path/to/config.json  # Path to authorization configuration file
  authz_reload_interval: 5m            # Interval to reload the configuration
```

## Authorization Configuration Format
The authorization configuration file should be in JSON format:

```json
[
  {
    "tenants": ["tenant1", "tenant2"],
    "serialNumber": ["01:23:45:67:89:ab:cd:ef"]
  },
  {
    "tenants": ["tenant3"],
    "serialNumber": ["fe:dc:ba:98:76:54:32:10"]
  }
]
```

This configuration maps TLS certificate serial numbers to allowed tenant IDs.

## Behavior
- When enabled, the frontend will validate that the client's TLS certificate serial number is authorized to access the requested tenant.
- Configuration is loaded at startup and automatically reloaded at the specified interval.
- If authorization is disabled, no certificate validation will be performed.