# Agent Context — pipeline-conversion-cost

## What This Repo Is

This is a working research and preparation repo for a conference talk titled:

> **The Invisible Tax: How Data Format Conversions Drive Up Telemetry Pipeline Costs**

See [README.md](README.md) for the full abstract.

## Who Is Working Here

The repo owner (cijothomas) is the speaker preparing this talk. The work here includes:
- Slide outlines and speaker notes in [slides.md](slides.md)
- Benchmarks and profiling experiments
- Code modifications to demonstrate and measure conversion costs
- Collector configurations for profiling

## Why the Cloned Repos Are Here

Several upstream OTel repos are cloned as subdirectories for direct reference, code reading, and running benchmarks. They are **excluded from git tracking** via [.gitignore](.gitignore).

| Directory | Repo | Why It's Here |
|---|---|---|
| `opentelemetry-rust/` | [open-telemetry/opentelemetry-rust](https://github.com/open-telemetry/opentelemetry-rust) | Running SDK-side benchmarks; modified `basic-otlp` example used as a load generator to send OTLP to the collector |
| `opentelemetry-dotnet/` | [open-telemetry/opentelemetry-dotnet](https://github.com/open-telemetry/opentelemetry-dotnet) | Comparing SDK-side serialization costs across languages; BenchmarkDotNet results |
| `opentelemetry-collector/` | [open-telemetry/opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector) | Reading collector internals: pdata model, receiver/exporter pipeline, proto decode path |
| `otel-arrow/` | [open-telemetry/otel-arrow](https://github.com/open-telemetry/otel-arrow) | Understanding Arrow-based pipeline: lazy decode, columnar format, passthrough without deserialization (Rust `otap-dataflow`) |

## Key Files

| File | Purpose |
|---|---|
| [slides.md](slides.md) | Talk outline + speaker notes. Contains detailed technical analysis, real benchmark data, and slide-by-slide content |
| [otel-collector/config.yaml](otel-collector/config.yaml) | Collector config with `pprof` extension for CPU profiling |
| [otel-collector/README.md](otel-collector/README.md) | How to run collector with profiling and capture flamegraphs |
| `cpu.prof` | Captured Go pprof CPU profile from a passthrough pipeline (OTLP in → debug exporter) |
| `opentelemetry-rust/opentelemetry-otlp/examples/basic-otlp/src/main.rs` | Modified to emit logs in an infinite loop — used as a load generator |

## Core Research Questions

1. **How much CPU does `proto.Unmarshal` consume in a passthrough collector pipeline?** (measured: ~25% of total CPU, receive side only)
2. **What does the full Go Collector decode path look like, function by function?**
3. **How does the OTel Arrow / OTAP approach differ** — specifically the lazy `OtapPayload::OtlpBytes` representation that avoids decode when no processor needs the data?
4. **What are the SDK-side serialization costs** in Rust and .NET?

## Profiling Setup

The local profiling environment:
- Collector running in Docker using `otel-collector/config.yaml` (pprof on port 1777)
- Rust `basic-otlp` example sending logs via OTLP gRPC to the collector in a tight loop
- CPU profiles captured with: `curl -o cpu.prof "http://localhost:1777/debug/pprof/profile?seconds=30"`
- Flamegraphs viewed with: `go tool pprof -http=:8080 cpu.prof`

## Tone and Approach for This Talk

- Audience: platform/infra engineers and OTel practitioners
- Goal: educate, not just criticize — show the problem, quantify it, and point toward solutions
- The Go Collector's behavior is not a bug, it is an architectural constraint — be precise about this
- OTel Arrow / OTAP is the forward-looking solution to highlight, based on actual code in `otel-arrow/`
