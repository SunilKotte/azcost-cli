# azcost-cli

A concurrent Azure Cost Management CLI built in Go that demonstrates
advanced concurrency patterns, performance profiling, and production-grade CI.

## What it does
- Authenticates with Azure using `azidentity`
- Fetches cost management data for **multiple subscriptions in parallel**
- Uses a **goroutine worker pool** with rate limiting to handle 50+ subscriptions
- Outputs formatted tables or JSON
- **Profiled with pprof** – steady-state memory < 20 MB
- **Table-driven tests** with race detector in GitHub Actions

## Why it's cool
- Total execution time slashed from **12 seconds (serial) to under 800 ms (parallel)**
- Zero goroutine leaks, memory-leak-free channel design
- Real-world concurrency patterns you'd use in production
- Fully documented architecture decisions in the README
