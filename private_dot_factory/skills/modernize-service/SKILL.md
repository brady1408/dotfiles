---
name: modernize-service
description: Refactor a Go service to remove navmon (Milieu) and deprecated serverd packages. Invoke with /modernize-service then provide the service name. Use when migrating services away from the legacy navmon god object to direct serverd package usage.
---

# Modernize Service

Remove `navmon.Milieu` and deprecated serverd package usage from a single Go service,
replacing with direct serverd package calls following the patterns in `exemplar-go/`.

## Inputs

The user provides a **service name** (a top-level directory in the monorepo, e.g. `account`, `unit-cache`, `transunion`).

## Instructions

### Step 1: Study the exemplar

Before making any changes, read these reference files to understand the target patterns:

1. `exemplar-go/main.go` -- canonical service bootstrap without navmon
2. `exemplar-go/internal/config/config.go` -- config struct using embedded serverd configs
3. `exemplar-go/internal/jobs/error_handler.go` -- River ErrorHandler with Snag (if service uses River)
4. `exemplar-go/internal/jobs/doc.go` -- River config example
5. `.factory/skills/modernize-service/references.md` -- full navmon method -> serverd mapping table
6. `.factory/skills/modernize-service/removals.md` -- packages to remove vs keep

### Step 2: Scan the service

Find all files in `<service>/` that import navmon or any removal-target package:

```
grep -r "libraries/navmon" <service>/ --include="*.go" -l
grep -r "serverd/navslack\|serverd/objstore\|serverd/navriver\|serverd/navencryption" <service>/ --include="*.go" -l
```

Create a todo list of every file that needs changes.

### Step 3: Understand the service's config

Read the service's config file (usually `internal/config/config.go` or `pkg/config/config.go`).
Understand which navmon fields the service actually uses -- many services only use a subset
(e.g. Database + Redis but not Vault or Slack).

### Step 4: Refactor the config

Replace the `navmon.Milieu` struct (or embedding of it) with a service-specific config struct
that embeds only the serverd config structs the service actually needs.

Follow the pattern in `exemplar-go/internal/config/config.go`:
- Embed `*navdb.DBPoolConfig` if using database
- Embed `grpc.GrpcConfig` if using gRPC server
- Embed `http.HTTPdConfig` if using HTTP server
- Add `navredis.RedisConfig` if using Redis
- Add `navvault.VaultConfig` if using Vault
- Add `navfeatureflag.LaunchDarklyConfig` if using feature flags
- Keep `AppName`, `LogLevel`, `Habitat` as direct fields

### Step 5: Refactor each file

For each file that imports navmon, use the mapping in `references.md` to replace:

1. **Config references**: Replace `m.DbHost`, `m.GRPCPort`, etc. with the appropriate embedded struct field
2. **Method calls**: Replace `m.Database()`, `m.Redis()`, etc. with direct serverd calls
3. **Bootstrap**: Replace `m.Bootstrap(ctx)` with explicit `navlifecycle.MustLifeCycleManager()` + `navotel.MustOtel()` setup
4. **Server setup**: Replace `m.ServeHTTP()` / `m.ServeGrpc()` with direct serverd server constructors
5. **Imports**: Remove the navmon import, add the specific serverd package imports

### Step 6: Remove deprecated packages

If the service imports any packages listed in `removals.md` as "always remove" or "remove if encountered":
- `navslack` / `objstore`: Simply remove dead imports
- `navriver`: Replace with direct `river.Config{}` usage (see `exemplar-go/internal/jobs/`)
- `navencryption`: Inline the ~30 lines into the service

### Step 7: Validate

Run the validation script to confirm everything compiles, lints, and tests pass:

```
./bin/validate <service>
```

Fix any issues that emerge. Common issues:
- Missing imports for new serverd packages
- Unused imports of navmon
- Type mismatches where Milieu fields were passed to functions expecting specific types
- BUILD.bazel files need updating (gazelle should handle this via `bazel run //:gazelle`)

### Step 8: Report

Summarize what was changed:
- Number of files modified
- Which navmon methods were replaced
- Which deprecated packages were removed
- Whether validation passed

## Important constraints

- Do NOT remove or modify `navlifecycle` or `navdb` usage -- these are staying
- Do NOT address "questionable" serverd packages (navredis, navhttp, navclients, etc.) -- those are future work
- Do NOT modify files outside the target service directory
- Do NOT batch multiple services -- one at a time, fully validated
- Always run `./bin/validate <service>` before reporting completion
- Preserve existing env variable names and defaults so deployed services don't break
