# Packages to Remove

When modernizing a service, remove usage of these packages entirely.

## Always remove

| Package | Consumers | Action |
|---------|-----------|--------|
| `libraries/navmon` | 25 services | Replace with direct serverd package usage per references.md |
| `libraries/serverd/navslack` | 0 outside libraries | Remove dead imports |
| `libraries/serverd/objstore` | 0 outside libraries | Remove dead imports |

## Remove if encountered in target service

| Package | Consumers | Action |
|---------|-----------|--------|
| `libraries/serverd/navriver` | 0 outside libraries | Replace with direct `river.Config{}` per exemplar-go/internal/jobs/ |
| `libraries/serverd/navencryption` | 1 (person) | Inline ~30 lines into the consuming service |

## Do NOT remove (even if they look like thin wrappers)

These packages are staying:
- `navlifecycle` -- LifecycleManager is the standard service orchestration pattern
- `navdb` and sub-packages -- standard DB layer
- `navlog` -- standard logging
- `naverror` -- standard error utilities
- `navid` -- standard ID generation
- `navinterceptors` -- standard gRPC header propagation
- `navfeatureflag` -- standard feature flag interface
- `navvault` -- standard Vault integration
- `navruntime` -- standard env/git helpers
- `navotel` -- standard OTel bootstrap
- `navgrpc` -- standard gRPC client constructor
- `navtokenizer` -- standard tokenizer client
- `pathformatter` -- standard URL path formatting for metrics
- `metrics` -- standard Prometheus helpers
- `acl` -- standard gRPC ACL
- `serverd/grpc` -- standard gRPC server
- `serverd/http` -- standard HTTP server
- `sfmc` -- Salesforce Marketing Cloud client
- `sfsync` -- Salesforce CRM client
