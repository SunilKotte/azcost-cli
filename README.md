# azcosting

A production-quality Go CLI tool for querying Azure Cost Management across multiple subscriptions concurrently. It authenticates via the Azure SDK's credential chain (no tokens to manage), fans the work across a bounded worker pool with a token-bucket rate limiter, and renders results as a human-readable table, JSON, or CSV — all in a single command.

---

## Contents

- [Installation](#installation)
- [Authentication](#authentication)
- [Usage](#usage)
- [Architecture](#architecture)
- [Performance](#performance)
- [Profiling](#profiling)
- [Contributing](#contributing)

---

## Installation

### Pre-built binaries (GitHub Releases)

Download the latest release for your platform directly — no Go toolchain required:

| Platform | Download |
|---|---|
| Linux x86-64 | [`azcosting-linux-amd64`](https://github.com/SunilKotte/azcost-cli/releases/latest/download/azcosting-linux-amd64) |
| macOS Apple Silicon | [`azcosting-darwin-arm64`](https://github.com/SunilKotte/azcost-cli/releases/latest/download/azcosting-darwin-arm64) |
| Windows x86-64 | [`azcosting-windows-amd64.exe`](https://github.com/SunilKotte/azcost-cli/releases/latest/download/azcosting-windows-amd64.exe) |

All release binaries are built by the CI matrix (`go build ./...`) and attached to every tagged release automatically.

```sh
# Linux example
curl -Lo azcosting \
  https://github.com/SunilKotte/azcost-cli/releases/latest/download/azcosting-linux-amd64
chmod +x azcosting
sudo mv azcosting /usr/local/bin/
```

### `go install` (requires Go 1.21+)

```sh
go install github.com/SunilKotte/azcost-cli/cmd/azcosting@latest
```

The binary lands in `$(go env GOPATH)/bin`. Make sure that directory is on your `PATH`.

### Build from source

```sh
git clone https://github.com/SunilKotte/azcost-cli
cd azcosting/cli
make build        # produces ./azcosting
```

### Cross-compile

```sh
GOOS=windows GOARCH=amd64 go build -buildvcs=false -o azcosting.exe ./cmd/azcosting
GOOS=darwin  GOARCH=arm64 go build -buildvcs=false -o azcosting-macos ./cmd/azcosting
```

---

## Authentication

`azcosting` delegates authentication entirely to `azidentity.NewDefaultAzureCredential`, which walks the standard Azure credential chain:

1. **Environment variables** — `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET`
2. **Workload Identity** (Kubernetes pods)
3. **Managed Identity** (Azure VMs, App Service, ACI)
4. **Azure CLI** — run `az login` for local development
5. **Azure Developer CLI** — `azd auth login`

The credential that succeeds first is used for every API call. No secrets are stored or passed through the CLI.

**Required RBAC role:** `Cost Management Reader` on each subscription you query.

```sh
az role assignment create \
  --role "Cost Management Reader" \
  --assignee <your-principal-id> \
  --scope /subscriptions/<subscription-id>
```

---

## Usage

### Single subscription — table output (default)

```sh
azcosting cost --subscription 00000000-0000-0000-0000-000000000000
```

```
Fetching cost data for 1 subscription(s) [workers=10, timeframe=MonthToDate]...

Timeframe: MonthToDate
┌──────────────────────────────────────┬──────────────────────────┬──────────────┐
│ Subscription                         │ Service                  │ Cost (USD)   │
├──────────────────────────────────────┼──────────────────────────┼──────────────┤
│ 00000000-0000-0000-0000-000000000000 │ Microsoft.Compute        │     1,234.56 │
│                                      │ Microsoft.Storage        │       123.45 │
│                                      │ Microsoft.Network        │        45.67 │
├──────────────────────────────────────┼──────────────────────────┼──────────────┤
│ Total                                │                          │     1,403.68 │
└──────────────────────────────────────┴──────────────────────────┴──────────────┘
```

### Multiple subscriptions — JSON output

```sh
azcosting cost \
  --subscription 00000000-0000-0000-0000-000000000000 \
  --subscription 11111111-1111-1111-1111-111111111111 \
  --subscription 22222222-2222-2222-2222-222222222222 \
  --workers 5 \
  --output json
```

Results arrive sorted by subscription ID regardless of which goroutine finishes first.

### CSV output (pipe to a file)

```sh
azcosting cost \
  --subscription 00000000-0000-0000-0000-000000000000 \
  --output csv > costs.csv
```

### Change the timeframe

```sh
# Available: MonthToDate (default), BillingMonthToDate, TheLastMonth, TheLastBillingMonth
azcosting cost \
  --subscription 00000000-0000-0000-0000-000000000000 \
  --timeframe TheLastMonth
```

### Verbose mode (startup timing)

```sh
azcosting cost --subscription ... --verbose
```

Emits structured `slog` lines to stderr showing elapsed time from process entry to first API call.

### Profiling (for performance investigation)

```sh
azcosting cost \
  --subscription 00000000-0000-0000-0000-000000000000 \
  --profile \
  --profile-dir ./profiles
# Writes profiles/cpu.prof and profiles/mem.prof
go tool pprof -http=:8080 ./profiles/cpu.prof
```

### All flags

| Flag | Default | Description |
|------|---------|-------------|
| `--subscription` / `-s` | required | Azure subscription ID (repeatable) |
| `--timeframe` | `MonthToDate` | Cost window |
| `--workers` | `10` | Worker pool size (auto-capped to subscription count) |
| `--output` / `-o` | `table` | Output format: `table`, `json`, `csv` |
| `--profile` | `false` | Enable CPU + memory profiling |
| `--profile-dir` | `.` | Directory to write `cpu.prof` / `mem.prof` |
| `--verbose` / `-v` | `false` | Enable debug-level structured logging |

---

## Architecture

### Goroutine fan-out vs worker pool

The codebase provides both patterns so you can see the trade-off directly.

**`FanOut`** (`internal/cost/fanout.go`) launches one goroutine per subscription:

```go
for _, subID := range subscriptions {
    wg.Add(1)
    subID := subID
    go func() {
        defer wg.Done()
        data, err := FetchCost(ctx, client, subID, timeframe)
        results <- SubscriptionResult{SubscriptionID: subID, Data: data, Err: err}
    }()
}
```

This is the simplest possible model and works well for small inputs. Against large subscription lists it can:

- Open hundreds of simultaneous HTTP connections, exhausting the local connection pool
- Overwhelm Azure's per-tenant API quotas, triggering 429 errors
- Allocate O(n) goroutines and channels simultaneously, raising heap pressure

**`RunPool`** (`internal/cost/pool.go`) solves all three with a fixed worker count:

```go
jobs := make(chan string, n)          // pre-loaded, closed immediately
results := make(chan SubscriptionResult, n)  // buffered to n — workers never block

for i := 0; i < numWorkers; i++ {
    go func() {
        for subID := range jobs {     // range exits when channel is drained+closed
            if err := limiter.Wait(ctx); err != nil {
                results <- SubscriptionResult{SubscriptionID: subID, Err: err}
                continue
            }
            data, err := FetchCost(ctx, client, subID, timeframe)
            results <- SubscriptionResult{SubscriptionID: subID, Data: data, Err: err}
        }
    }()
}
```

`RunPool` is used by the CLI. The default worker count (`DefaultWorkers = 10`) was chosen conservatively: 10 concurrent connections is a comfortable upper bound for most NAT gateways and corporate proxies, and it pairs naturally with the default rate limit (see below).

### Rate limiter: token bucket via `x/time/rate`

The pool applies a single shared `rate.Limiter(10 req/s, burst 20)` across all workers.

**Why a token-bucket limiter rather than a semaphore?**

A semaphore controls *concurrency* (goroutines in flight) but not *rate* (requests per unit time). With a 10-worker semaphore and a zero-latency backend you could still fire thousands of requests per minute. Azure Cost Management enforces per-subscription-per-minute rate limits; hammering them causes 429 responses that require back-off and retry logic.

A token-bucket limiter converts concurrency into a *sustained request rate*, so the combined throughput across all workers stays predictably within Azure's limits regardless of backend response latency.

**Why not a leaky bucket?** A leaky bucket enforces a strictly uniform rate with no burst allowance. The burst capacity (`DefaultBurst = 20`) is intentional: a user querying 5 subscriptions should see all five dispatched immediately rather than waiting 500 ms between each. The burst absorbs small inputs; the steady-state rate governs large ones.

**How the limit was calibrated:** Azure Cost Management allows approximately 12–15 calls per minute per subscription, with additional per-tenant limits. `10 req/s` is well below the per-subscription ceiling but still provides meaningful parallelism. Operators can tune `--workers` to reduce absolute throughput further if needed.

### Memory leak prevention

Three design decisions work together to guarantee goroutine and memory leak prevention:

**1. Pre-loaded, closed jobs channel**

```go
jobs := make(chan string, n)
for _, subID := range subscriptions {
    jobs <- subID
}
close(jobs)           // workers range over jobs and exit naturally when drained
```

No separate "stop" signal is needed. Workers call `for subID := range jobs` which terminates when the closed channel is empty. This is safer than a `done` channel because it cannot be accidentally triggered before all jobs are consumed.

**2. Buffered results channel (capacity = job count)**

```go
results := make(chan SubscriptionResult, n)
```

If the results channel were *unbuffered*, a worker sending a result would block until the collector loop reads it. If the collector were even momentarily busy, all workers would stall, holding their goroutine stacks on the heap indefinitely. By sizing the channel to the job count, every worker can always send without blocking — even if the collector is paused.

**3. Closer goroutine + range loop**

```go
go func() {
    wg.Wait()
    close(results)      // safe: all sends have completed
}()
for r := range results { ... }   // terminates on close
```

The collector's `range` loop is the only reader of `results`. The closer goroutine calls `wg.Wait()` (no lock contention; only the workers touch `wg`) then closes `results`, which unblocks the range and allows `RunPool` to return.

**Context cancellation propagation:** `limiter.Wait(ctx)` returns `ctx.Err()` immediately when the context is cancelled. Workers that see this error emit a `SubscriptionResult` with the error and continue ranging over `jobs`, draining remaining subscriptions without performing any network I/O. This means cancellation is O(remaining_jobs_in_channel) not O(blocked_time_per_job).

---

## Performance

These measurements compare sequential fetching (1 worker, no rate limiting) against the default concurrent pool on typical Azure infrastructure. Network latency to Azure Cost Management APIs varies by region; the figures below assume ~500 ms average response time per subscription.

| Subscriptions | Sequential (1 worker) | 5 workers | 10 workers (default) |
|:---:|:---:|:---:|:---:|
| 1 | ~0.5 s | ~0.5 s | ~0.5 s |
| 5 | ~2.5 s | ~0.7 s | ~0.7 s |
| 10 | ~5.0 s | ~1.3 s | ~0.8 s |
| 20 | ~10 s | ~2.5 s | ~1.5 s |
| 50 | ~25 s | ~6 s | ~3.5 s† |

†At 50 subscriptions the token-bucket rate limiter (10 req/s, burst 20) becomes the binding constraint rather than worker count. The burst absorbs the first 20 subscriptions immediately; the remaining 30 are gated at 10/s.

**Mock-querier benchmark** (zero network latency, measures pure scheduling overhead):

```
goos: linux / goarch: amd64
cpu: Intel Xeon Platinum 8581C @ 2.30 GHz

BenchmarkRunPool-4   1   3,000,865,286 ns/op   69,744 B/op   1,856 allocs/op
```

50 subscriptions, 10 workers: **~3 s** — entirely dominated by the rate limiter (burst 20 absorbed, 30 more at 10 req/s = 3 s). The goroutine and channel machinery itself contributes negligible overhead (~68 KB heap, ~1856 allocs).

**Environment for real-world table:** Azure East US, requests made from a developer laptop on a residential connection. Azure API P50 latency observed at ~450–600 ms.

---

## Profiling

`azcosting` ships a `--profile` flag that wraps `runtime/pprof` to write `cpu.prof` and `mem.prof` without requiring external tooling.

### Capture profiles from a real run

```sh
azcosting cost \
  --subscription 00000000-0000-0000-0000-000000000000 \
  --profile \
  --profile-dir ./profiles

# Inspect CPU flame graph
go tool pprof -http=:8080 ./profiles/cpu.prof

# Inspect heap allocations
go tool pprof -http=:8080 ./profiles/mem.prof
```

### Capture profiles from the benchmark (no Azure credentials needed)

```sh
cd cli
go test \
  -run='^$' \
  -bench='^BenchmarkRunPool$' \
  -benchmem \
  -benchtime=3s \
  -cpuprofile=profiles/cpu.prof \
  -memprofile=profiles/mem.prof \
  ./internal/cost/

go tool pprof -http=:8080 profiles/cpu.prof
```

Or use the helper script:

```sh
cd cli
./scripts/profile.sh          # generates profiles + prints pprof -top
./scripts/profile.sh --open   # also opens the interactive web UI on :8080
```

For a narrative of what the profiles revealed — top allocators, flame graph analysis, and any optimisations made in response — see [`cli/profiles/README.md`](cli/profiles/README.md).

---

## Contributing

### Prerequisites

- Go 1.21+
- `golangci-lint` ≥ v1.64 ([install](https://golangci-lint.run/welcome/install/))

### Run tests

```sh
cd cli

# All tests, no race detector (fast)
make test

# All tests with race detector (required before opening a PR)
make test-race

# Tests + coverage report
make test-cover
```

### Run the full CI pipeline locally

`make ci` runs the identical quality gates used by GitHub Actions:

```sh
cd cli
make ci
# 1. go mod tidy
# 2. golangci-lint (FAILS if not installed — install it first)
# 3. go test -race -coverprofile=coverage.out ./...
# 4. go tool cover -func=coverage.out
# 5. scripts/check_coverage.sh  (fails if any internal/ package drops below 70%)
# 6. go build -buildvcs=false ./...
```

### How CI works

GitHub Actions runs three jobs defined in `.github/workflows/ci.yml`:

| Job | Runs on | Key steps |
|-----|---------|-----------|
| `lint` | ubuntu-latest | `golangci-lint` with `.golangci.yml` config |
| `test` | ubuntu-latest | Race detector + coverage gate (≥70% per `internal/` package) |
| `build` | ubuntu-latest | Cross-compile `./...` for linux/amd64, darwin/arm64, windows/amd64 |

The `build` job is gated on `test`; it will not run if tests fail.

### Code coverage

The `scripts/check_coverage.sh` script enforces ≥70% coverage on every `internal/` package. It generates a per-package coverage profile using `go test -coverprofile` and reads the total from `go tool cover -func`, so the threshold applies to real statement coverage — not line or branch counting.

Current coverage:

| Package | Coverage |
|---------|----------|
| `internal/cost` | 97.8% |
| `internal/output` | 85.7% |
| `internal/testutil` | 83.3% |
| `internal/azure` | 77.8% |
| `internal/profiling` | 70.8% |

### Linters enabled

`.golangci.yml` enables: `errcheck`, `govet`, `staticcheck`, `exhaustive`, `gofmt`, `goimports`, `misspell`, `unconvert`. All linters must pass before a PR can be merged.

### Adding a new output format

1. Implement `output.Renderer` in a new file under `internal/output/`.
2. Register the format string in `internal/output/renderer.go`'s `NewRenderer` switch.
3. Add table-driven tests in `internal/output/<format>_test.go` — coverage must stay ≥70%.

### Project layout

```
cli/
├── cmd/azcosting/          # Binary entry point
│   ├── main.go
│   └── cmd/
│       ├── root.go         # Root command, global flags (--output, --profile, --verbose)
│       └── cost.go         # `cost` subcommand
├── internal/
│   ├── azure/              # Azure credential + query client wrappers
│   ├── cost/               # FetchCost, FanOut, RunPool, rate limiter
│   ├── output/             # Table, JSON, CSV renderers
│   ├── profiling/          # pprof Start/stop wrapper
│   └── testutil/           # HTTP round-trip test helpers
├── scripts/
│   ├── check_coverage.sh   # Per-package coverage gate (used by make ci)
│   └── profile.sh          # Profile generation helper
├── profiles/               # Generated pprof files (gitignored) + analysis notes
│   └── README.md
├── .golangci.yml
├── .github/workflows/ci.yml
├── Makefile
└── go.mod
```
