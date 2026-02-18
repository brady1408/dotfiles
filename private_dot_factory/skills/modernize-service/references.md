# navmon -> serverd Migration Reference

## Method Mapping

| navmon usage | Replacement | Import |
|---|---|---|
| `navmon.MustLoad()` / `Milieu` struct | Individual config structs per package (see exemplar-go/internal/config/) | Remove `navmon` import entirely |
| `m.Bootstrap(ctx)` | `navotel.MustOtel()` + `navlog.SetLevel()` + `signal.NotifyContext()` | `serverd/navotel`, `serverd/navlog` |
| `m.Database()` | `navdb.DBPoolConfig{...}.MustDBPool(ctx, logger, lcm)` | `serverd/navdb` |
| `m.Redis(ctx)` | `navredis.RedisConfig{...}.MustRedis(ctx, logger, lcm)` | `serverd/navredis` |
| `m.VaultInstance(ctx)` | `navvault.VaultConfig{...}.MustVault(ctx, logger, lcm)` | `serverd/navvault` |
| `m.FlaggerInstance()` | `navfeatureflag.MustNew(ctx, logger, &cfg, lcm)` | `serverd/navfeatureflag` |
| `m.Slack(ctx)` | `navslack.SlackConfig{...}.MustSlack(ctx)` | `serverd/navslack` |
| `m.MustReachardRsaKey()` | `navencryption.RSAConfig{...}.MustReachardRsaKey()` | `serverd/navencryption` |
| `m.MustClientCreds()` | `navtils.TLSConfig{...}.MustClientCreds()` | `serverd/navtils` |
| `m.ServeHTTP(ctx, mux, pf)` | `http.HttpServer(cfg, logger, appName).WithHandler(mux).WithPathFormatter(pf)` + `lcm.AddServiceToManage(httpServer)` + `lcm.RunService(ctx, logger)` | `serverd/serverd/http` |
| `m.ServeGrpc(ni)` | `navgrpc.GrpcdServer(cfg, logger)` | `serverd/serverd/grpc` |
| `m.GrpcClient(svc, dial, creds)` | `metrics.GrpcClient(appName, svc, dial, creds)` | `serverd/metrics` |
| `m.Counter(prefix)` | `metrics.Prometheus{}.Counter(prefix)` | `serverd/metrics` |
| `m.CounterVec(prefix, labels...)` | `metrics.Prometheus{}.CounterVec(prefix, labels...)` | `serverd/metrics` |
| `m.GaugeVec(prefix, labels...)` | `metrics.Prometheus{}.GaugeVec(prefix, labels...)` | `serverd/metrics` |
| `m.MeteredClient(prefix, pf)` | `metrics.Prometheus{}.MeteredClient(prefix, http.DefaultTransport, pf, nil)` | `serverd/metrics` |
| `m.MeteredTransportClient(prefix, rt, pf, timeout)` | `metrics.Prometheus{}.MeteredClient(prefix, rt, pf, timeout)` | `serverd/metrics` |
| `m.MustLimiter(ctx)` | `navredis.RedisConfig{...}.MustRedisLimiter(ctx, logger, lcm)` | `serverd/navredis` |
| `m.Bugsnag(ctx, secrets...)` | `navbugsnag.BootstrapBugsnag(ctx)` | `serverd/navbugsnag` |
| `m.InEnv()` | `navruntime.InEnv()` | `serverd/navruntime` |
| `m.AppName` | Read from config struct field or `os.Getenv("CONFIGMAPNAME")` | stdlib |
| `m.Habitat` | Read from config struct field or `os.Getenv("HABITAT")` | stdlib |
| `m.LogLevel` | Read from config struct field or `os.Getenv("LOG_LEVEL")` | stdlib |
| `m.GRPCPort` | Use `grpc.GrpcConfig` embedded struct | `serverd/serverd/grpc` |
| `m.HTTPPort` | Use `http.HTTPdConfig` embedded struct | `serverd/serverd/http` |
| `m.DbHost/DbName/DbPort/...` | Use `navdb.DBPoolConfig` embedded struct | `serverd/navdb` |
| `m.RedisUrl/RedisRetries/...` | Use `navredis.RedisConfig` struct | `serverd/navredis` |
| `m.VaultURL/VaultAuthSource` | Use `navvault.VaultConfig` struct | `serverd/navvault` |

## Config struct pattern

Replace the Milieu struct with a service-specific config that embeds the individual serverd config structs.
See `exemplar-go/internal/config/config.go` for the canonical example:

```go
package config

import (
    "github.com/nav-inc/go/libraries/serverd/navdb"
    "github.com/nav-inc/go/libraries/serverd/serverd/grpc"
    "github.com/nav-inc/go/libraries/serverd/serverd/http"
)

type Config struct {
    AppName             string `env:"CONFIGMAPNAME" default:"CONFIGMAPNAME-is-not-set"`
    LogLevel            string `env:"LOG_LEVEL" default:"info"`
    *navdb.DBPoolConfig
    grpc.GrpcConfig
    http.HTTPdConfig
    // Add other config structs as needed:
    // navredis.RedisConfig
    // navvault.VaultConfig
    // navfeatureflag.LaunchDarklyConfig
}
```

## Bootstrap pattern

Replace `m.Bootstrap(ctx)` with explicit setup. See `exemplar-go/main.go` for the canonical example:

```go
logger := navlifecycle.GetLogger()
lcm := navlifecycle.MustLifeCycleManager(ctx, logger, conf.LogLevel, conf.AppName)
defer lcm.Shutdown(ctx)
```

## River pattern

Use `river.Config{}` directly. See `exemplar-go/internal/jobs/` for ErrorHandler with Snag.
See `billing-microservice/cmd/river.go` for a full working example of River worker setup.

```go
riverCfg := &river.Config{
    Workers:      workers,
    Queues:       map[string]river.QueueConfig{"default": {MaxWorkers: 5}},
    ErrorHandler: &jobs.ErrorHandler{},
    JobTimeout:   10 * time.Second,
    MaxAttempts:  5,
    Logger:       slog.Default(),
}
client, err := river.NewClient(pool.RiverDriver(), riverCfg)
```
