# OTel Collector — Profiling Setup

Run a local OpenTelemetry Collector with the `pprof` extension enabled to measure real CPU and memory costs of a passthrough pipeline.

---

## Run the Collector

### Option A: Docker (easiest)

```bash
docker run --rm \
  -v $(pwd)/config.yaml:/etc/otelcol/config.yaml \
  -p 4317:4317 \
  -p 4318:4318 \
  -p 1777:1777 \
  -p 13133:13133 \
  otel/opentelemetry-collector-contrib:latest \
  --config /etc/otelcol/config.yaml
```

### Option B: Binary

Download from https://github.com/open-telemetry/opentelemetry-collector-releases/releases

```bash
./otelcol-contrib --config config.yaml
```

---

## Capture Profiles

The `pprof` extension exposes standard Go pprof endpoints at `http://localhost:1777/debug/pprof/`.

### CPU Profile (30-second capture while sending load)

```bash
# Capture a 30-second CPU profile
curl -o cpu.prof "http://localhost:1777/debug/pprof/profile?seconds=30"

# Analyze interactively
go tool pprof -http=:8080 cpu.prof
```

### Heap / Memory Profile

```bash
curl -o heap.prof "http://localhost:1777/debug/pprof/heap"
go tool pprof -http=:8080 heap.prof
```

### Goroutine Profile

```bash
curl -o goroutine.prof "http://localhost:1777/debug/pprof/goroutine"
go tool pprof -http=:8080 goroutine.prof
```

### Trace (for detailed timeline)

```bash
curl -o trace.out "http://localhost:1777/debug/pprof/trace?seconds=10"
go tool trace trace.out
```

---

## Send Load to the Collector

Use `telemetrygen` (OTel load generator) to generate synthetic traffic:

```bash
# Install
go install github.com/open-telemetry/opentelemetry-collector-contrib/cmd/telemetrygen@latest

# Send traces
telemetrygen traces --otlp-insecure --rate 1000 --duration 60s

# Send metrics
telemetrygen metrics --otlp-insecure --rate 1000 --duration 60s

# Send logs
telemetrygen logs --otlp-insecure --rate 1000 --duration 60s
```

Or use the OTLP HTTP endpoint (port 4318) from your application directly.

---

## What to Look For in the Profile

When analyzing the CPU profile, look for hot functions related to format conversion:

| Hot function pattern | What it means |
|---|---|
| `proto.Marshal` / `proto.Unmarshal` | Protobuf serialization cost |
| `pdata` package functions | Collector internal model conversion |
| `encoding/gzip` | Compression overhead |
| `grpc` / `http` transport | Wire protocol overhead |
| Your processor logic | Legitimate value-add work |

The goal: quantify what fraction of CPU is **pure conversion** (marshal/unmarshal/copy) versus **value-generating processing**.

---

## Passthrough Baseline Experiment

To isolate pure conversion cost with no processing, edit `config.yaml` and remove the `batch` processor:

```yaml
processors: []  # no processors = measure raw conversion only
```

Compare CPU profiles between:
1. No processors (pure conversion baseline)
2. With `batch` processor (realistic minimal pipeline)
3. With additional processors (e.g., `attributes`, `transform`)

This shows exactly how much CPU is burned before any real work happens.

---

## Useful Links

- [pprof extension docs](https://github.com/open-telemetry/opentelemetry-collector/tree/main/extension/pprofextension)
- [OTel Collector releases](https://github.com/open-telemetry/opentelemetry-collector-releases/releases)
- [telemetrygen](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/cmd/telemetrygen)
- [Go pprof docs](https://pkg.go.dev/net/http/pprof)
