# The Invisible Tax: How data format conversions drive up telemetry pipeline costs

Telemetry signals traverse long pipelines before reaching observability backends. While enrichment, filtering, and redaction provide clear value, significant compute cost often comes from repeated conversion through different data formats.

Telemetry commonly flows through SDK formats, wire protocols, collector‑internal formats, and backend ingestion schemas. Each boundary introduces marshaling, unmarshalling and copying. These transformations add no new information, yet consume CPU and memory and scale linearly with volume—creating a hidden "transform tax" that compounds dramatically at terabyte scale.

This talk will share results from measuring instrumented OpenTelemetry SDK and Collector pipelines. We quantify compute spent on pure format conversion versus value‑generating processing and show how these costs grow with scale.

Attendees will learn about conversion costs and strategies to reduce waste: eliminating unnecessary translations, aligning pipeline representations, leveraging zero‑copy techniques, and minimizing transformation hops between pipeline stages. We also examine Apache Arrow‑based representations as one approach to reducing this overhead.
