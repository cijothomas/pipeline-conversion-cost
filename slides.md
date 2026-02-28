# Slides Outline: The Invisible Tax — How Data Format Conversions Drive Up Telemetry Pipeline Costs

**Total: 40 min talk + 10 min Q&A**

---

## Slide 1 — Title
- The Invisible Tax: How Data Format Conversions Drive Up Telemetry Pipeline Costs
- Speaker name / event / date

---

## Slide 2 — Who Is This For?
- Platform/infra engineers running telemetry pipelines
- Anyone paying unexpected cloud bills for observability
- OTel users curious about what happens "under the hood"

---

## Slide 3 — The Hook (~2 min)
- "You're paying for processing you never asked for"
- Teaser: at terabyte scale, format conversion alone can dominate CPU usage
- What if we could measure exactly how much?

---

## Section 1: Defining the Pipeline (~5 min)

### Slide 4 — What Is a Telemetry Pipeline?
- Starts the moment a log/metric/trace is **emitted** via OTel SDK or library
- Ends when the data is **stored and queryable** in a backend
- Everything in between = the pipeline

### Slide 5 — Pipeline Stages (diagram)
```
App (OTel SDK)
  → Exporter (OTLP/gRPC or HTTP)
  → Collector (receive → process → export)
  → Collector (optional chained)
  → Ingestion layer (backend-specific)
  → Storage / Query engine
```
- Each arrow = a format boundary

### Slide 6 — Pipeline Has Real Value
- Enrichment, filtering, redaction, sampling
- Routing to multiple backends
- Aggregation and transformation
- These justify the pipeline's existence

---

## Section 2: The Hidden Transform Tax (~7 min)

### Slide 7 — What Happens at Each Boundary?
- Marshal → transmit → unmarshal → copy into internal representation
- Repeat at every stage
- No new information is created

### Slide 8 — The Format Zoo
- OTel SDK internal model (language-specific types)
- OTLP Protobuf (wire format)
- Collector pdata (internal in-memory model)
- Backend-specific ingestion format
- Storage format (Parquet, columnar, etc.)

### Slide 9 — The "Transform Tax" Defined
- CPU cycles spent on pure format conversion
- Memory allocations for intermediate copies
- Scales **linearly with volume**
- Compounds across pipeline stages
- Invisible in standard observability dashboards

---

## Section 3: SDK-Side Costs (~10 min)

### Slide 10 — How OTel SDK Produces Data
- User calls SDK → data captured in SDK-internal model
- SDK serializes to OTLP Protobuf for export
- Brief walk through the SDK export path

### Slide 11 — What Does Serialization Cost?
- Benchmark: SDK export path profiling
- CPU breakdown: business logic vs. serialization
- Memory allocations per span/metric/log

### Slide 12 — .NET SDK Measurements
- Benchmarks from opentelemetry-dotnet
- [insert numbers: ns/op, allocations, % time in serialization]
- Source: BenchmarkDotNet results

### Slide 13 — Rust SDK Measurements
- Benchmarks from opentelemetry-rust
- [insert numbers: ns/op, % time in serialization]
- Source: Criterion benchmarks

### Slide 14 — SDK-Side Takeaways
- Serialization is non-trivial even before data leaves the process
- Proto encoding + field copying = measurable overhead
- Amplified when SDK → Collector is on the same host (unnecessary wire serialization)

---

## Section 4: Collector-Side Costs (~10 min)

### Slide 15 — Inside the OTel Collector
- Receiver: deserialize OTLP → pdata
- Processor(s): operate on pdata
- Exporter: serialize pdata → OTLP (or other format)
- Every hop: deserialize + copy + serialize

### Slide 16 — pdata: Collector's Internal Model
- Wrapper types around Protobuf-generated structs
- Designed for correctness and extensibility
- Not designed for zero-copy or column-oriented access

### Slide 17 — Measuring Collector Conversion Cost
- Instrumented pipeline: receive → passthrough → export (no processing)
- CPU profile of what a "no-op" pipeline actually does
- [insert benchmark numbers: MB/s throughput, CPU % for conversion]

### Slide 18 — Chained Collectors Make It Worse
```
Collector A: receive OTLP → pdata → export OTLP
Collector B: receive OTLP → pdata → export OTLP
```
- Each collector re-deserializes and re-serializes
- N collectors = N × conversion cost
- Common in enterprise deployments (agent → gateway → cloud)

### Slide 19 — Collector-Side Takeaways
- Even a passthrough pipeline burns significant CPU
- Chaining multiplies cost
- Processors add value; conversions do not

---

## Section 5: Ingestion & Beyond (Brief Mention) (~3 min)

### Slide 20 — What Happens After the Collector?
- Ingestion layer: OTLP → backend's native format
- Storage layer: row → columnar (e.g., Parquet)
- Query layer: may re-hydrate/decode again
- Not measuring these today, but the pattern continues

---

## Section 6: Strategies to Reduce the Tax (~5 min)

### Slide 21 — Strategy 1: Eliminate Unnecessary Hops
- Do you need that second collector in the chain?
- SDK → backend direct where appropriate
- Fewer stages = fewer conversions

### Slide 22 — Strategy 2: Align Representations
- Keep data in the same format across stages
- Avoid format round-trips for pass-through data

### Slide 23 — Strategy 3: Zero-Copy Techniques
- Pass references, not copies
- Avoid re-serializing data that isn't modified

### Slide 24 — Strategy 4: OTel Arrow
- Apache Arrow as a columnar, zero-copy-friendly representation
- Batch many spans/metrics together → amortize overhead
- Arrow Flight for transport: avoid repeated (de)serialization
- [brief demo or benchmark comparison]

### Slide 25 — OTel Arrow: What Changes?
- Replace OTLP Protobuf with Arrow IPC on the wire
- Collector can process without full deserialization for passthrough
- Dramatic reduction in CPU for high-volume pipelines

---

## Section 7: Closing (~3 min)

### Slide 26 — What We Measured
- SDK: X% of export time is serialization (not instrumentation)
- Collector: Y% of CPU in a passthrough pipeline is conversion
- At 1 TB/day, this translates to ~$Z/month in wasted compute (estimated)

### Slide 27 — Key Takeaways
1. The telemetry pipeline is longer than you think
2. Format conversion is a silent, scalable cost
3. Quantify before you optimize
4. OTel Arrow is a promising path for high-volume pipelines

### Slide 28 — Call to Action
- Profile your own pipeline
- Contribute benchmarks to opentelemetry-dotnet / opentelemetry-rust
- Try OTel Arrow exporter in OpenTelemetry Collector
- Links / resources

### Slide 29 — Thank You / Q&A
- Repo / slides link
- Contact / social

---

## Timing Summary

| Section | Slides | Time |
|---|---|---|
| Hook | 1–3 | ~2 min |
| Pipeline Definition | 4–6 | ~5 min |
| Transform Tax | 7–9 | ~7 min |
| SDK-Side Costs | 10–14 | ~10 min |
| Collector-Side Costs | 15–19 | ~10 min |
| Ingestion Mention | 20 | ~3 min |
| Strategies + Arrow | 21–25 | ~5 min |
| Closing | 26–29 | ~3 min |
| **Total** | | **~40 min** |

---

## Open TODOs / Numbers to Fill In
- [ ] Actual benchmark numbers from opentelemetry-dotnet BenchmarkDotNet runs
- [ ] Actual benchmark numbers from opentelemetry-rust Criterion runs  
- [ ] Collector passthrough pipeline CPU profile numbers
- [ ] OTel Arrow vs OTLP comparison benchmark
- [ ] Cost estimate: translate CPU overhead to $/month at scale
