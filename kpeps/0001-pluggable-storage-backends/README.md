# KPEP-0001: Pluggable storage backends

| Field | Value |
|---|---|
| **Status** | `provisional` |
| **Authors** | @zachsmith |
| **Created** | 2026-05-29 |
| **Last updated** | 2026-06-02 |
| **Tracking issue** | TBD |
| **Affected repos** | `kplane-dev/storage` (registry, in-tree backends), `kplane-dev/apiserver`, (archived) `kplane-dev/spanner` |
| **Supersedes** | N/A |

## Summary

Today the kplane apiserver picks its storage backend with a hardcoded `if opts.SpannerProject != ""` branch in `cmd/apiserver/app/config.go`. The Spanner experiment proved that direct `storage.Interface` implementation is the right contract (vs. shimming through etcd via kine), but the wiring shortcut doesn't scale to multiple backends.

This KPEP graduates that experiment to an instance-scoped registry of named backends, sitting *above* the upstream `storagebackend.factory.Create` switch. The fork is not modified. The CLI surface mirrors upstream exactly: `--storage-backend=<name>` (already an upstream flag, currently restricted to `"etcd3"`) selects the backend, and each backend brings its own `--<name>-*` flag block the same way `--etcd-*` flags belong to etcd. Backend factories match upstream's `factory.Create` signature so they slot into the existing per-resource decoder pipeline.

## Motivation

The Spanner experiment did its job. It proved direct `storage.Interface` implementation works, and gave us `kplane-dev/spanner` (824 lines) as a worked example with the right signature: `BackendFactory func(*storagebackend.ConfigForResource, newFunc, newListFunc func() runtime.Object, resourcePrefix string) (storage.Interface, factory.DestroyFunc, error)`, 1-to-1 with upstream's `factory.Create`. The wiring around it was the throwaway part: a `SpannerConfig` struct in `multicluster.Options` and an `if opts.SpannerProject != ""` branch in the apiserver. The `--spanner-*` flags themselves are fine and idiomatic; what isn't is the apiserver knowing Spanner's name.

Two pieces of evidence push us toward generalizing now:

1. **kplane-kine (`kplane-dev/kplane-kine`)** confirms kine isn't a universal answer: 1.5–3× lower throughput, 2–14× wider p99, and a 1s watch floor on the polling driver. Some deployments will trade that for postgres operability; others won't. We'll want at least three backends (Spanner, kine→postgres for ergonomics-first deployments, postgres-native for perf-sensitive ones), possibly more.
2. **The experiment's seams are showing.** Adding a second backend with today's pattern means another option struct and another `if` branch in `config.go`. External consumers can't add a backend at all without forking the apiserver.

Cost of not graduating now:
- Every new backend is an apiserver patch. The apiserver becomes the choke point for backend experimentation.
- The contract a backend has to honor stays implicit, only readable by reverse-engineering `spanner/store.go`. The next backend will get subtle parts wrong (watch ordering, RV monotonicity, compaction) and we'll find out in production.

### Goals

1. Any backend implementing `storage.Interface` can be added by writing a Go package and one `backends.Register(...)` line. No apiserver code change required for in-tree backends; one `main.go` line for out-of-tree.
2. Backend factories reuse the existing upstream `factory.Create` signature so they slot into the per-resource decoder pipeline without contract reinvention.
3. The apiserver selects a backend via the existing upstream `--storage-backend=<name>` flag. Each backend owns its own flag block, prefixed by the backend name, exactly the way `--etcd-*` flags belong to etcd today.
4. Etcd is a registered backend in the registry, not a special case. The default path (`--storage-backend=etcd3` or unset) goes through the same lookup as every other backend.
5. A conformance suite proves any registered backend satisfies the contract upstream `storage.Interface` consumers expect. It extends `k8s.io/apiserver/pkg/storage/testing` (the existing upstream test utilities) rather than reimplementing them.
6. The Spanner experiment migrates into the new location as the first non-etcd registered backend, behavior unchanged. `--spanner-*` flags remain canonical (not deprecated).

### Non-goals

1. **Modifying the upstream fork.** Upstream rejected pluggable storage years ago (kubernetes/kubernetes#1957); kubernetes/enhancements#172 has been in limbo for ~8 years. The fork is exemplary minimal today (2 commits, 15 files, all about multicluster identity). Patching `storagebackend/factory/factory.go` would add permanent rebase tax across 5 parallel switches (`Create`, `CreateHealthCheck`, `CreateReadyCheck`, `CreateProber`, `CreateMonitor`) for zero upstream value.
2. **A YAML config file for backend configuration.** Upstream doesn't use one for storage; only for things that are structured (encryption providers per resource, admission plugin chains, structured auth config). Storage is "one backend per process," which fits naturally as CLI flags. The encryption-provider-config precedent is a red herring once we look at how `--etcd-*` is actually shaped.
3. **Writing a postgres-native backend.** Obvious next backend after the registry lands, but separate KPEP.
4. **Replacing kine.** kine stays a valid choice; this KPEP makes it one option among many rather than the only non-etcd path.
5. **Changing `storage.Interface` itself.** We honor the upstream contract as-is.
6. **Hot reload of backend config.** Backend swaps imply tearing down watchers and reconnecting; restart is the honest answer.
7. **Multi-backend per apiserver process.** One apiserver, one backend. Per-resource routing is a future option.
8. **The per-cluster service IP allocator bug** (tracked separately at `kplane-dev/infrastructure#8`). Orthogonal.

## Proposal

The registry is an instance-scoped `Backends` struct in `kplane-dev/storage/registry`, mirroring upstream's `admission.Plugins`. It's not a package-level global: the apiserver constructs one in `main`, the in-tree backends register against it via a single aggregator file, and the apiserver hands it to the options/config build path.

Each backend implements a `Backend` interface that mirrors upstream's options-struct lifecycle (`AddFlags`, `Validate`, then build):

```go
// kplane-dev/storage/registry/registry.go
package registry

import (
    "sync"

    "github.com/spf13/pflag"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apiserver/pkg/storage"
    "k8s.io/apiserver/pkg/storage/storagebackend"
    "k8s.io/apiserver/pkg/storage/storagebackend/factory"
)

// Factory matches upstream storagebackend/factory.Create exactly so backends
// slot into the existing per-resource decoder pipeline. Called once per
// resource at REST registry construction time.
type Factory func(
    config *storagebackend.ConfigForResource,
    newFunc, newListFunc func() runtime.Object,
    resourcePrefix string,
) (storage.Interface, factory.DestroyFunc, error)

// Backend is the contract every registered backend implements. The lifecycle
// matches upstream's options-struct convention: bind flags, validate after
// parse, build the per-resource factory once.
type Backend interface {
    // Name is the value matched against --storage-backend.
    Name() string

    // AddFlags binds the backend's CLI flags to fs. Called for every
    // registered backend at startup so all flags appear in --help.
    AddFlags(fs *pflag.FlagSet)

    // Validate runs the backend's flag-level validation. Called only for the
    // selected backend, after flag parsing.
    Validate() []error

    // Build returns the per-resource Factory. Called once after Validate.
    Build() (Factory, error)
}

type Backends struct {
    mu         sync.RWMutex
    registered map[string]Backend
}

func New() *Backends
func (b *Backends) Register(backend Backend)        // panic on duplicate name
func (b *Backends) Get(name string) (Backend, bool)
func (b *Backends) Names() []string                 // sorted, for help + validation
func (b *Backends) AddFlags(fs *pflag.FlagSet)      // iterates all registered
```

Lifecycle: at apiserver startup, the registry is constructed and every backend registered. `Backends.AddFlags(fs)` binds *all* backends' flags so `--help` shows everything (`--etcd-*` and `--spanner-*` and future `--postgres-*` all visible). After flag parse, the apiserver looks up the selected backend by `--storage-backend`, calls its `Validate`, then its `Build`, and threads the resulting `Factory` into the existing `DecoratorConfig.BackendFactory` field. The registry is then idle.

### User stories

#### Story 1: An operator deploys kplane on Spanner

```
kube-apiserver \
  --storage-backend=spanner \
  --spanner-project=my-project \
  --spanner-instance=my-instance \
  --spanner-database=kplane \
  --spanner-credentials-file=/etc/kplane/spanner-sa.json \
  ...
```

The flag set looks exactly like an etcd deployment, just with a different prefix. `kube-apiserver --help` lists every registered backend's flag block; the operator picks one by name.

#### Story 2: Adding the postgres-native backend (in-tree)

An engineer adds `kplane-dev/storage/backends/postgres/`:

1. Write `backend.go` implementing `storage.Interface` with `LISTEN/NOTIFY` for watch.
2. Write `options.go` exporting an `Options` struct that implements `registry.Backend` (`Name()`, `AddFlags`, `Validate`, `Build`).
3. Add one line to `kplane-dev/storage/backends/register.go`:
   ```go
   b.Register(postgres.NewOptions())
   ```
4. Run the conformance suite (`go test ./conformance/... -backend=postgres -postgres-dsn=...`). Iterate until green.

The apiserver doesn't change. The flag set gains `--postgres-*` entries automatically. Operators opt in with `--storage-backend=postgres`.

#### Story 3: An external organization adds a backend

A third party adds `github.com/some-org/kplane-cockroach`. Their `Options` struct implements `registry.Backend`. To deploy:

1. They build a custom apiserver `main.go` that adds one line after the in-tree registrations:
   ```go
   backends := registry.New()
   storagebackends.RegisterBuiltin(backends)
   backends.Register(cockroach.NewOptions())  // their addition
   ```
2. They run it with `--storage-backend=cockroach --cockroach-dsn=...`.

No fork of `kplane-dev/storage` required. This matches how third-party admission plugins extend the apiserver today.

### Notes / constraints / caveats

- **Factory contract is upstream-stable.** We honor `factory.Create`'s signature so any backend works with the existing per-resource decoder, the watch cache, and the storage decorator chain that our multicluster identity hooks (`WrapDecodedObject`, etc.) thread through.
- **The registry sits at `kplane-dev/storage/registry`** as a subpackage to keep clean references; `kplane-dev/storage` is already `package storage` and importing `k8s.io/apiserver/pkg/storage` from a same-named package, while legal, is noisy.
- **`--storage-backend` is already upstream.** It's currently validated against `{"etcd3"}` (a hardcoded set in upstream `EtcdOptions.Validate`). Our apiserver's options validation widens the accepted set to `backends.Names()` and dispatches to the selected backend's `Validate`. The upstream flag binding itself is reused as-is.
- **All backend flags are always exposed.** `--etcd-servers` is in `--help` even when `--storage-backend=spanner`, and vice versa. This matches how upstream handles `--encryption-provider-config`, `--audit-policy-file`, and the admission plugin flags: bind everything, dispatch at runtime. Flag groups (via pflag's `FlagSet`) can section the help output by backend.
- **Etcd is a registered backend.** `kplane-dev/storage/backends/etcd/` is a thin wrapper that owns its own `--etcd-*` flag bindings and a `storagebackend.Config`, and whose `Build` returns a `Factory` calling upstream `factory.Create`. Cross-cutting storage-layer flags (`--storage-media-type`, `--watch-cache`, `--encryption-provider-config`, etc.) stay on upstream `EtcdOptions` (now a misnomer, but ours to move only if we fork) and are orthogonal to backend selection.
- **Spanner experiment migrates entirely.** `kplane-dev/spanner` is archived or left as a one-release re-export shim. The duplicate `BackendFactory` type currently in `kplane-dev/spanner/factory.go` ("to avoid an import cycle") becomes `registry.Factory`.

## Design details

### Repo layout

```
kplane-dev/storage/
  registry/
    registry.go              Backend interface, Backends, Factory; instance-scoped
  conformance/
    suite.go                 Run(t, Backend{...}); extends k8s.io/.../storage/testing
    multicluster_test.go     identity hooks, key isolation, our additions
  backends/
    register.go              RegisterBuiltin(b *registry.Backends) aggregator
    etcd/
      options.go             Options struct; --etcd-* flags; Build delegates to factory.Create
    spanner/
      backend.go             migrated from kplane-dev/spanner (store, broadcaster)
      options.go             Options struct; --spanner-* flags
      factory.go             returns registry.Factory
      conformance_test.go    runs conformance.Run against the Spanner emulator
    postgres/                future
      ...

kplane-dev/apiserver/
  cmd/apiserver/main.go      constructs registry, calls backends.RegisterBuiltin(b)
  cmd/apiserver/app/options/
    options.go               aggregates backends.AddFlags + --storage-backend dispatch
  cmd/apiserver/app/config.go  resolves selected Backend, calls Validate, Build, threads Factory
  pkg/multicluster/storage.go  takes a resolved registry.Factory; no per-backend branches
```

External backends live in their own repos and are added by an operator's custom apiserver `main.go` calling `backends.Register(<their>.NewOptions())`.

### Registry interface

```go
// kplane-dev/storage/registry/registry.go
package registry

import (
    "fmt"
    "sort"
    "sync"

    "github.com/spf13/pflag"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apiserver/pkg/storage"
    "k8s.io/apiserver/pkg/storage/storagebackend"
    "k8s.io/apiserver/pkg/storage/storagebackend/factory"
)

type Factory func(
    config *storagebackend.ConfigForResource,
    newFunc, newListFunc func() runtime.Object,
    resourcePrefix string,
) (storage.Interface, factory.DestroyFunc, error)

type Backend interface {
    Name() string
    AddFlags(fs *pflag.FlagSet)
    Validate() []error
    Build() (Factory, error)
}

type Backends struct {
    mu         sync.RWMutex
    registered map[string]Backend
}

func New() *Backends {
    return &Backends{registered: map[string]Backend{}}
}

func (b *Backends) Register(backend Backend) {
    name := backend.Name()
    b.mu.Lock()
    defer b.mu.Unlock()
    if _, exists := b.registered[name]; exists {
        panic(fmt.Sprintf("storage backend %q already registered", name))
    }
    b.registered[name] = backend
}

func (b *Backends) Get(name string) (Backend, bool) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    backend, ok := b.registered[name]
    return backend, ok
}

func (b *Backends) Names() []string {
    b.mu.RLock()
    defer b.mu.RUnlock()
    names := make([]string, 0, len(b.registered))
    for n := range b.registered {
        names = append(names, n)
    }
    sort.Strings(names)
    return names
}

func (b *Backends) AddFlags(fs *pflag.FlagSet) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    for _, backend := range b.registered {
        backend.AddFlags(fs)
    }
}
```

### Per-backend wiring

```go
// kplane-dev/storage/backends/spanner/options.go
package spanner

import (
    "context"
    "fmt"

    "github.com/spf13/pflag"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apiserver/pkg/storage"
    "k8s.io/apiserver/pkg/storage/storagebackend"
    "k8s.io/apiserver/pkg/storage/storagebackend/factory"

    "kplane-dev/storage/registry"
)

type Options struct {
    Project         string
    Instance        string
    Database        string
    EmulatorHost    string
    CredentialsFile string
}

func NewOptions() *Options { return &Options{} }

func (o *Options) Name() string { return "spanner" }

func (o *Options) AddFlags(fs *pflag.FlagSet) {
    fs.StringVar(&o.Project, "spanner-project", o.Project,
        "Google Cloud project hosting the Spanner instance.")
    fs.StringVar(&o.Instance, "spanner-instance", o.Instance,
        "Spanner instance ID.")
    fs.StringVar(&o.Database, "spanner-database", o.Database,
        "Spanner database name within the instance.")
    fs.StringVar(&o.EmulatorHost, "spanner-emulator-host", o.EmulatorHost,
        "Optional: address of a local Spanner emulator (host:port). When set, the project/instance/database flags still apply but credentials are skipped.")
    fs.StringVar(&o.CredentialsFile, "spanner-credentials-file", o.CredentialsFile,
        "Path to a Google service account JSON file.")
}

func (o *Options) Validate() []error {
    var errs []error
    if o.Project == "" {
        errs = append(errs, fmt.Errorf("--spanner-project is required when --storage-backend=spanner"))
    }
    if o.Instance == "" {
        errs = append(errs, fmt.Errorf("--spanner-instance is required when --storage-backend=spanner"))
    }
    if o.Database == "" {
        errs = append(errs, fmt.Errorf("--spanner-database is required when --storage-backend=spanner"))
    }
    // credentials are optional when EmulatorHost is set
    return errs
}

func (o *Options) Build() (registry.Factory, error) {
    // Dial once, share the client across every per-resource Factory call.
    // The shared client is not explicitly closed; it relies on process exit
    // to drop the session pool, which matches what `apiserver/pkg/multicluster/
    // storage.go:455-501` does today (sync.Once-guarded init, no Close).
    // The per-resource DestroyFunc handles per-resource teardown only
    // (e.g., stopping the broadcaster goroutine for that resource's watch).
    client, err := newClient(context.Background(), o)
    if err != nil {
        return nil, fmt.Errorf("dialing spanner: %w", err)
    }

    return func(
        rfc *storagebackend.ConfigForResource,
        newFunc, newListFunc func() runtime.Object,
        resourcePrefix string,
    ) (storage.Interface, factory.DestroyFunc, error) {
        store, stopBroadcaster := newStore(client, rfc, newFunc, newListFunc, resourcePrefix)
        return store, stopBroadcaster, nil
    }, nil
}
```

The etcd wrapper has the same shape, but its `Build` delegates to upstream:

```go
// kplane-dev/storage/backends/etcd/options.go
package etcd

import (
    "github.com/spf13/pflag"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apiserver/pkg/storage"
    "k8s.io/apiserver/pkg/storage/storagebackend"
    "k8s.io/apiserver/pkg/storage/storagebackend/factory"

    "kplane-dev/storage/registry"
)

type Options struct {
    config storagebackend.Config  // upstream's typed config
}

func NewOptions() *Options {
    return &Options{config: *storagebackend.NewDefaultConfig("/registry", nil)}
}

func (o *Options) Name() string { return "etcd3" }

func (o *Options) AddFlags(fs *pflag.FlagSet) {
    fs.StringSliceVar(&o.config.Transport.ServerList, "etcd-servers", o.config.Transport.ServerList,
        "List of etcd servers to connect with (scheme://ip:port), comma separated.")
    fs.StringVar(&o.config.Prefix, "etcd-prefix", o.config.Prefix,
        "The prefix to prepend to all resource paths in etcd.")
    fs.StringVar(&o.config.Transport.KeyFile, "etcd-keyfile", o.config.Transport.KeyFile,
        "SSL key file used to secure etcd communication.")
    fs.StringVar(&o.config.Transport.CertFile, "etcd-certfile", o.config.Transport.CertFile,
        "SSL certification file used to secure etcd communication.")
    fs.StringVar(&o.config.Transport.TrustedCAFile, "etcd-cafile", o.config.Transport.TrustedCAFile,
        "SSL CA file used to secure etcd communication.")
    // ... timing / health flags ...
}

func (o *Options) Validate() []error {
    var errs []error
    if len(o.config.Transport.ServerList) == 0 {
        errs = append(errs, fmt.Errorf("--etcd-servers must be specified when --storage-backend=etcd3"))
    }
    return errs
}

func (o *Options) Build() (registry.Factory, error) {
    cfg := o.config  // copy
    return func(
        rfc *storagebackend.ConfigForResource,
        newFunc, newListFunc func() runtime.Object,
        resourcePrefix string,
    ) (storage.Interface, factory.DestroyFunc, error) {
        // Merge our connection config into the per-resource config the
        // apiserver provides (which carries codec, transformer, prefix, etc.),
        // then delegate to upstream factory.Create.
        merged := *rfc
        merged.Transport = cfg.Transport
        if merged.Prefix == "" {
            merged.Prefix = cfg.Prefix
        }
        return factory.Create(merged, newFunc, newListFunc, resourcePrefix)
    }, nil
}
```

The aggregator:

```go
// kplane-dev/storage/backends/register.go
package backends

import (
    "kplane-dev/storage/backends/etcd"
    "kplane-dev/storage/backends/spanner"
    // "kplane-dev/storage/backends/postgres"  // future
    "kplane-dev/storage/registry"
)

// RegisterBuiltin installs the in-tree backends into b. External backends
// call b.Register(<theirs>.NewOptions()) from a custom apiserver main.
func RegisterBuiltin(b *registry.Backends) {
    b.Register(etcd.NewOptions())
    b.Register(spanner.NewOptions())
    // b.Register(postgres.NewOptions())
}
```

### Apiserver wiring

The apiserver constructs the registry once, binds all backend flags before parse, then dispatches at validate/build time.

```go
// kplane-dev/apiserver/cmd/apiserver/main.go
import (
    "kplane-dev/storage/registry"
    storagebackends "kplane-dev/storage/backends"
)

func main() {
    backends := registry.New()
    storagebackends.RegisterBuiltin(backends)
    // (External backends would be added here in a custom build.)

    opts := options.NewOptions(backends)
    cmd := app.NewServerCommand(opts)
    // ...
}
```

```go
// kplane-dev/apiserver/cmd/apiserver/app/options/options.go
type Options struct {
    // ... existing fields ...

    Backends *registry.Backends
}

func NewOptions(backends *registry.Backends) *Options {
    return &Options{
        // ...
        Backends: backends,
    }
}

func (o *Options) AddFlags(fs *pflag.FlagSet) {
    // ... existing flag bindings ...
    // --storage-backend is already bound by upstream EtcdOptions.AddFlags.
    o.Backends.AddFlags(fs)  // adds every backend's flag block
}

func (o *Options) Validate() []error {
    var errs []error
    // Widen upstream's hardcoded {"etcd3"} validation.
    selected := o.Etcd.StorageConfig.Type
    if selected == "" {
        selected = "etcd3"
    }
    backend, ok := o.Backends.Get(selected)
    if !ok {
        errs = append(errs, fmt.Errorf("--storage-backend=%q invalid, allowed values: %v", selected, o.Backends.Names()))
    } else {
        errs = append(errs, backend.Validate()...)
    }
    // ... other existing validation, but skip upstream EtcdOptions.Validate's
    // etcd-specific checks when a non-etcd backend is selected ...
    return errs
}
```

```go
// kplane-dev/apiserver/cmd/apiserver/app/config.go
selected := opts.Etcd.StorageConfig.Type
if selected == "" {
    selected = "etcd3"
}
backend, _ := opts.Backends.Get(selected)
backendFactory, err := backend.Build()
if err != nil {
    return nil, err
}
mcOpts.BackendFactory = backendFactory
// The hardcoded `if opts.SpannerProject != ""` branch goes away entirely.
```

The existing `--spanner-*` flags continue to work; their bindings move from the apiserver's options package into `kplane-dev/storage/backends/spanner/options.go`. There is no deprecation period and no flag rename because nothing operator-visible changes.

### Conformance suite

Upstream already has `k8s.io/apiserver/pkg/storage/testing` with `TestCreate`, `TestGet`, `TestWatch`, `TestList`, `TestGuaranteedUpdate`, etc. that it uses to test etcd3. The conformance suite imports these and adds the cases not covered:

```go
// kplane-dev/storage/conformance/suite.go
package conformance

import (
    "testing"
    "k8s.io/apiserver/pkg/storage"
    storagetesting "k8s.io/apiserver/pkg/storage/testing"
)

// Harness is what the backend's test provides: a function that produces a
// fresh storage.Interface plus the cleanup that tears it back down. The
// harness is responsible for constructing a storagebackend.ConfigForResource
// (codec, versioner, prefix, etc.); the conformance suite supplies the
// per-test object types.
type Harness func(t *testing.T) (storage.Interface, func())

type Backend struct {
    Name string
    New  Harness
}

func Run(t *testing.T, b Backend) {
    t.Run("upstream", func(t *testing.T) { runUpstreamSuite(t, b, storagetesting.RunTests) })
    t.Run("multicluster/identity", func(t *testing.T) { testIdentity(t, b) })
    t.Run("multicluster/keyIsolation", func(t *testing.T) { testKeyIsolation(t, b) })
    t.Run("watch/ordering-under-concurrent-writes", func(t *testing.T) { testWatchOrdering(t, b) })
}
```

Each backend's repo has a `conformance_test.go` that calls `conformance.Run`. That's the merge gate for a backend.

### Test plan

- **Conformance suite** is the contract proof. Every backend's CI runs it.
- **Spanner CI** (now in `backends/spanner/`) runs conformance against the Spanner emulator.
- **Etcd CI** (in `backends/etcd/`) runs conformance against an in-process etcd. Catches regressions in our wrapper.
- **Apiserver CI** runs an end-to-end smoke against at least one registered backend (Spanner emulator) to prove the registry plumbing + flag binding + decorator threading work end-to-end.
- **kplane-kine** continues its own e2e + bench against kine; if we ever wrap kine as a registered backend, it also gets conformance.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Conformance suite under-specifies the contract; backends pass but diverge in production (watch ordering, RV gaps). | Seed from `k8s.io/apiserver/pkg/storage/testing` (what etcd3 is held to). Add a regression every time we find a backend bug; the suite is a living artifact. |
| Hidden coupling in current Spanner code (the apiserver assumes Spanner-shaped state somewhere we haven't found). | The Spanner migration is the canary: anything that breaks is something we'd hit when adding the second backend anyway. Better to find it now. |
| Flag explosion: `--help` lists every backend's flag block whether used or not. | Matches upstream's pattern for admission/encryption/auth. Mitigate readability with pflag flag groups so each backend's flags render in their own help section. Document that only the selected backend's flags take effect. |
| Upstream `EtcdOptions.Validate()` checks `--etcd-servers` unconditionally, which would falsely fail when `--storage-backend=spanner`. | Our `Options.Validate` skips upstream's etcd-specific validation when a non-etcd backend is selected, and routes to the selected backend's `Validate()` instead. Upstream `EtcdOptions` continues to own cross-cutting flags (`--storage-media-type`, `--watch-cache`, etc.) that don't depend on backend choice. |
| The `etcd` backend wrapper duplicates upstream's flag definitions for `--etcd-*`. | Real cost, small (~10 flag bindings in one file). Pays for itself by making etcd a first-class backend in the registry instead of a special case in dispatch logic. |

## Alternatives considered

### Alternative A: Modify the fork to make `storagebackend/factory.Create` pluggable

Replace the `switch` in `vendor/k8s.io/apiserver/pkg/storage/storagebackend/factory/factory.go` with a registered map, add a `Register` function, and migrate `CreateHealthCheck`, `CreateReadyCheck`, `CreateProber`, `CreateMonitor` to the same pattern.

**Rejected because:**
- The fork is intentionally minimal (2 commits, 15 files, all about multicluster identity). Modifying `factory.go` breaks the minimalism rule. Five parallel switches means five rebase points each upstream release.
- The upstream community has explicitly rejected pluggable storage (kubernetes/kubernetes#1957). Enhancement #172 ("alternative storage backend") has been stuck in `provisional` for ~8 years. The KEP candidacy framing for this approach is dead on arrival; SIG API Machinery's stated position is "use kine."
- This proposal sits above the fork at the `RESTOptionsGetter` / `StorageDecorator` layer, which is where our multicluster work *already* lives. We get the same outcome with zero fork modification.

### Alternative B: Single `--storage-config=<file.yaml>` configuration file

Define a `StorageConfiguration` YAML schema with per-backend typed blocks (or `RawExtension`). The apiserver loads it at startup; the registry dispatches the named block to the selected backend's parser. Mirrors `--encryption-provider-config`.

**Rejected because:** upstream doesn't use this pattern for storage. The etcd backend uses CLI flags (`--etcd-servers`, `--etcd-prefix`, etc.); we'd be introducing a parallel config-file system just for non-etcd backends, asymmetric and surprising to operators. Encryption uses a config file because it carries structured per-resource provider lists; storage is one-backend-per-process and fits naturally as flags. CLI flags also stay closer to the existing `--spanner-*` shape we already have, which makes the migration a rename-and-relocate rather than a re-platforming.

### Alternative C: `runtime.Scheme`-versioned config per backend

Each backend ships internal + external types, conversion, defaults, validation, and generated code (deepcopy/conversion/defaulter). Matches the `eventratelimit` admission plugin pattern.

**Rejected because:** ~700 LOC and 10 files per backend type. Upstream itself only uses this when shipping a durable user-facing API surface (kubelet config, scheduler config). For per-plugin internal config, upstream uses plain Go structs (`audit/buffered.BatchConfig` is the closest analog). With CLI flags we don't even need plain structs to be `runtime.Object`-shaped; they're just options structs.

### Alternative D: Use kine for everything non-etcd

Adopt kine as the only non-etcd path. Anything we want to add becomes a kine driver.

**Rejected because:** kplane-kine's bench shows kine's polling-watch floor (1s default, ~100ms minimum at idle DB cost), and the LogStructured layer forces architectural choices that prevent postgres `LISTEN/NOTIFY`. Kine is a great fit for some deployments; making it the *only* fit means accepting its perf shape platform-wide. The Spanner experiment proves direct `storage.Interface` implementation works and pays off.

### Alternative E: Global package-level registry (database/sql style)

Backends call `registry.Register("spanner", ...)` from `init()`; backends self-import via blank import `_ "kplane-dev/storage/backends/spanner"`.

**Rejected because:** upstream Kubernetes deliberately uses instance-scoped registries (`admission.Plugins`) over package globals. Tests construct fresh instances. Multiple coexisting registries are possible. Side-effect imports are harder to discover and to test. The instance-scoped pattern costs one extra `New()` call at startup.

### Alternative F: Special-case etcd; non-etcd backends go through the registry

If `--storage-backend=etcd3` or unset, use upstream's existing path (today's default) directly. Only non-etcd backends go through the registry. Avoids wrapping `EtcdOptions`.

**Rejected because:** asymmetric. The dispatch logic has two paths, the conformance suite has two harness shapes, and adding etcd-specific behavior to either path means thinking about both. The wrapper duplication (~10 flag bindings) is a one-time cost that buys consistent abstraction.

## Implementation plan

### MVP

- `kplane-dev/storage/registry/`: `Backend`, `Backends`, `Factory`, registry methods.
- `kplane-dev/storage/backends/etcd/`: wrapper owning `--etcd-*` flags; `Build` calls upstream `factory.Create`.
- `kplane-dev/storage/backends/spanner/`: migrated from `kplane-dev/spanner`. Owns `--spanner-*` flags (relocated from the apiserver).
- `kplane-dev/storage/backends/register.go`: `RegisterBuiltin` aggregator.
- `kplane-dev/apiserver`: registry construction in `main`, `--storage-backend` dispatch in options Validate, `Build` call in `app/config.go`. Hardcoded `if opts.SpannerProject != ""` branch removed.
- `kplane-dev/spanner`: archived (or thin re-export shim for one release).

This is enough to prove the pattern: same Spanner behavior, identical operator UX (`--spanner-*` flags unchanged), but the apiserver no longer knows Spanner's name.

### v1

- Conformance suite under `kplane-dev/storage/conformance/`. Spanner and etcd backends use it as their merge gate.
- pflag flag groups so `--help` renders backend flags in their own sections.

### Future / out of scope

- `kplane-dev/storage/backends/postgres/` with `LISTEN/NOTIFY` (separate KPEP).
- Possibly wrapping kine as a registered backend so kplane-kine deployments use the same flag surface.
- Per-resource backend routing (events on backend A, everything else on B).

## Open questions

1. **Where do cross-cutting storage flags live?** Upstream `EtcdOptions` currently owns `--storage-media-type`, `--watch-cache`, `--encryption-provider-config`, etc., which are orthogonal to backend choice. Keep them on upstream `EtcdOptions` (simplest, but the naming is misleading), move them to a new `StorageOptions` struct on the apiserver side (cleaner, but more wiring), or wrap upstream `EtcdOptions` in a thin shim (compromise). Decide at MVP-PR time.

2. **Does the etcd backend run upstream's `EtcdOptions.AddFlags` at all, or does it bind everything itself?** Running upstream's `AddFlags` pulls in cross-cutting flags as a side effect (see Q1), which we may or may not want. Binding everything ourselves means the etcd backend wrapper is ~30 flags instead of ~10. Choice tied to Q1.

## Drawbacks

- **One more layer.** A reader tracing "where does the apiserver get its storage" follows `main` → `RegisterBuiltin` → `Get` → `Build` → `Factory` instead of seeing a literal `NewSpannerStore(...)` call. Mitigated by the registry being ~80 lines of obvious code and the aggregator file listing every supported backend.
- **Flag explosion in `--help`.** Operators see every backend's flag block whether they use it or not. Matches upstream's behavior for admission/encryption/auth, but `kube-apiserver --help` is already 200+ lines and we're adding to it. pflag flag groups mitigate.
- **Etcd wrapper duplicates upstream's `--etcd-*` flag bindings.** ~10 flag declarations in one file. Real cost, small. Pays for itself by keeping etcd in the same abstraction as every other backend.
- **The conformance suite is a long-tail commitment.** Every contract subtlety we discover becomes a new test, and every backend has to keep passing. This is desired (we *want* that gate) but not free.

## History

- 2026-05-29: Initial draft (@zachsmith).
- 2026-06-02: Reworked against research findings (@zachsmith). Path A (modifying fork) moved to rejected alternatives. Factory signature aligned with upstream `factory.Create`. Registry instance-scoped (admission-style), not global. Backends collocated under `kplane-dev/storage/backends/<name>/`.
- 2026-06-02 (later): Configuration shape switched from YAML config file (`--storage-config`) to per-backend CLI flags, matching upstream's `--etcd-*` precedent. Etcd promoted to a registered backend (`backends/etcd/`) instead of a special case. External-backend story switched from forking the in-tree aggregator to extending a custom apiserver `main`.
