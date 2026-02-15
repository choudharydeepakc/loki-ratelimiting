# Add Rate Limiting and mTLS Authorization to Loki Frontend

## What this PR does / why we need it

This PR adds two optional features to the Loki query frontend:

1. **Redis-based rate limiting** - Protects against query flooding and provides per-tenant rate limits
2. **mTLS certificate-based authorization** - Enables secure multi-tenant access control using client certificates

Both features are disabled by default and maintain full backward compatibility.

## Which issue(s) this PR fixes

Addresses the need for:
- Query rate limiting to prevent resource exhaustion
- Certificate-based authorization for multi-tenant deployments
- Protection against noisy neighbor problems in shared environments

## Changes

### Rate Limiting (`pkg/lokifrontend/frontend/transport/ratelimit_adapter.go`)

Integrates with Redis to track rate limits across distributed Loki instances. Key features:
- Per-tenant rate limiting using `X-Scope-OrgID` header
- Fallback to IP-based limiting if tenant ID not available
- Request count and byte-based limits
- Standard HTTP 429 responses with `Retry-After` headers
- Dynamic tenant configuration via runtime API

**Implementation:**
```go
type RateLimitAdapter struct {
    cfg     RateLimitConfig
    service *ratelimit.Service
    redis   *redis.Client
    logger  log.Logger
}
```

The adapter wraps our go-ratelimit SDK and integrates it as middleware in the handler chain.

### Authorization (`pkg/lokifrontend/frontend/transport/handler.go`)

Validates client certificates and enforces tenant access policies. Key features:
- Extracts tenant ID from certificate CN (Common Name) field
- Validates OU (Organizational Unit) field format: `orgUnit:qualifier`
- Maps certificate serial numbers to allowed tenants via JSON config
- Hot-reloadable configuration (no restart required)
- Thread-safe with RWMutex protection

**Certificate parsing:**
```go
func parseSubject(subject string) (commonName, orgUnit, ouQualifier string) {
    // Extracts CN and splits OU on ":" 
    // Returns tenant ID (CN) and validates OU format
}
```

Both Subject and Issuer fields are parsed. A certificate is valid only if CN, orgUnit, and ouQualifier are all present.

## Configuration

### Rate Limiting
```yaml
frontend:
  rate_limit:
    enable: true
    redis_address: "redis:6379"
    redis_password: ""
    default_request_limit: 1000
    default_byte_limit: 104857600  # 100MB
    default_window: 60s
    rate_limit_by_tenant: true
    fallback_to_ip: true
```

### Authorization
```yaml
frontend:
  authz_enabled: true
  authz_config_path: "/etc/loki/authz-config.json"
  authz_reload_interval: 5m
```

**Authorization config file** (`/etc/loki/authz-config.json`):
```json
[
  {
    "serialNumber": ["01:23:45:67:89:ab:cd:ef"],
    "tenants": ["tenant-prod", "tenant-dev"]
  }
]
```

Maps certificate serial numbers to allowed tenant IDs.

## How to test

### Rate Limiting

1. Deploy Redis:
```bash
docker run -d --name redis -p 6379:6379 redis:7-alpine
```

2. Configure Loki with rate limiting enabled (see config above)

3. Send queries and verify rate limiting:
```bash
# Should work for first 100 requests
for i in {1..100}; do
  curl -H "X-Scope-OrgID: test" http://loki:3100/loki/api/v1/query?query={job="app"}
done

# Should return 429
curl -H "X-Scope-OrgID: test" http://loki:3100/loki/api/v1/query?query={job="app"}
```

Expected response headers:
- `X-RateLimit-Limit: 100`
- `X-RateLimit-Remaining: 0`
- `Retry-After: 45`

### Authorization

1. Generate client certificates with proper fields:
```
Subject: CN=tenant-prod, OU=dept-123:qualifier-456
```

2. Create authorization config mapping serial numbers to tenants

3. Configure Loki for mTLS:
```yaml
server:
  http_tls_config:
    cert_file: /path/to/server.crt
    key_file: /path/to/server.key
    client_auth_type: RequireAndVerifyClientCert
    client_ca_file: /path/to/ca.crt
```

4. Test authorization:
```bash
# Should succeed
curl --cert client.crt --key client.key \
     -H "X-Scope-OrgID: tenant-prod" \
     https://loki:3100/loki/api/v1/query?query={job="app"}

# Should fail with 401
curl --cert client.crt --key client.key \
     -H "X-Scope-OrgID: tenant-other" \
     https://loki:3100/loki/api/v1/query?query={job="app"}
```

Expected logs:
```
level=info msg="Authorization check passed" tenant=tenant-prod certTenantID=tenant-prod
level=debug msg="Authorization failed - tenant mismatch" certTenantID=tenant-prod requestedTenant=tenant-other
```

## Request flow

When both features are enabled:
1. **Rate limit check** - Returns 429 if limit exceeded
2. **Authorization check** - Returns 401 if unauthorized
3. **Process query** - Returns 200 if successful

```go
func (f *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Rate limiting (if enabled)
    if f.rateLimitAdapter != nil {
        middleware := f.rateLimitAdapter.QueryAPIMiddleware()
        handler := middleware(http.HandlerFunc(f.handleRequest))
        handler.ServeHTTP(w, r)
        return
    }
    f.handleRequest(w, r)
}

func (f *Handler) handleRequest(w http.ResponseWriter, r *http.Request) {
    // Authorization (if enabled)
    if f.cfg.AuthZEnabled {
        // Validate certificate and tenant
        if !authorized {
            http.Error(w, "unauthorized", 401)
            return
        }
    }
    // Process query...
}
```

## Performance impact

- **Rate limiting**: ~1-2ms overhead per request (Redis roundtrip)
- **Authorization**: <1ms overhead per request (in-memory map lookup)
- **Redis load**: Minimal (simple GET/SET/EXPIRE operations)
- **Memory**: ~10KB per 100 certificate mappings

## Special notes for reviewers


### Security considerations
- Rate limiting requires Redis on a private network
- Authorization config file should be readable only by Loki process
- Certificate serial numbers appear in debug logs
- Consider certificate expiry monitoring

### Dependencies
Adds:
- `github.com/redis/go-redis/v9` - Redis client
- `github.com/NVIDIA/go-ratelimitt` - Rate limiting SDK

### Backward compatibility
Both features are optional and disabled by default. Existing deployments work without any configuration changes.

### Future work
- Admin API for dynamic rate limit updates
- Prometheus metrics for rate limit violations
- RBAC support for authorization
- Integration with external identity providers
- Certificate revocation list (CRL) support

## Testing

Unit tests added in:
- `handler_ratelimit_test.go` - Rate limiting tests
- `handler_test.go` - Authorization tests

Run tests:
```bash
go test -v ./pkg/lokifrontend/frontend/transport -run TestRateLimit
go test -v ./pkg/lokifrontend/frontend/transport -run TestAuthZ
```

## Deployment notes

### Enabling rate limiting (no downtime)
1. Deploy Redis
2. Update Loki config with rate limiting disabled
3. Rolling restart
4. Enable with high limits
5. Gradually reduce to desired values

### Enabling authorization (requires planning)
⚠️ Requires mTLS setup and client certificate distribution. Test thoroughly in staging first.

---

**Questions?** 
