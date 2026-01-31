# spring-ai-opentelemetry-langfuse-observability
This solution provides a runnable Spring Boot setup that instruments Spring AI with OpenTelemetry and exports traces to a self-hosted Langfuse stack.

# LLM Observability for Spring AI: OpenTelemetry Tracing to Langfuse (Self-Hosted)

---

## 1) Problem
LLM-enabled services built with Spring AI lack first-class, production-grade observability. Teams cannot reliably trace prompt lifecycles, token usage, latency, or downstream failures across distributed systems. This creates blind spots in debugging, cost control, and compliance.
Teams struggle to answer basic operational questions:

- Which prompts and responses caused latency spikes?
- Where are tokens and cost accumulating across chains, tools, and retries?
- How do LLM calls correlate with upstream HTTP requests and downstream database activity?
- How can traces be inspected without sending sensitive data to third-party SaaS?

Standard APM tools capture HTTP spans but miss LLM-specific context (prompt, model, tokens, cost). Without structured tracing, root-cause analysis and cost governance are impractical.

---
View [Full runnable solution here](https://exesolution.com/solutions/spring-ai-opentelemetry-langfuse-observability)
## 2) Solution Overview (trace flow: app → OTEL → Langfuse)

This solution provides a runnable Spring Boot setup that instruments Spring AI with OpenTelemetry and exports traces to a **self-hosted Langfuse** stack.

**Trace flow:**

1. **Spring Boot application**  
   - Spring AI model calls (chat/completions, embeddings, tool calls) are wrapped with OpenTelemetry spans.
   - LLM metadata (model, token counts, latency, errors) is attached as span attributes.

2. **OpenTelemetry SDK (Java)**  
   - Context propagation links HTTP requests, business logic, and LLM calls into a single trace.
   - Sampling, batching, and redaction are enforced at the SDK level.

3. **OTLP Exporter**  
   - Spans are exported via OTLP (HTTP/gRPC) to the Langfuse ingestion endpoint.

4. **Langfuse (self-hosted)**  
   - Traces are persisted to PostgreSQL.
   - Engineers inspect prompts, responses, latency, cost, and errors in the Langfuse UI.

---

## 3) Runnable Implementation

### Project structure
```
.
├── app/
│ ├── src/main/java/... # Spring Boot + Spring AI application
│ ├── src/main/resources/
│ │ └── application.yml
│ └── Dockerfile
├── langfuse/
│ ├── docker-compose.yml # Langfuse + PostgreSQL
│ └── env.example
├── docker-compose.yml # Full stack (app + Langfuse)
└── evidence/
├── logs/
├── screenshots/
└── verification-checklist.md
```

The ZIP contains a complete, runnable project. No external services are required.


### Configuration (env vars + OTEL exporter settings)

**Application (Spring Boot):**
```
PRING_PROFILES_ACTIVE=otel
SPRING_AI_OPENAI_API_KEY=local-dev-key
```
**OpenTelemetry (Java SDK):**
```
OTEL_SERVICE_NAME=spring-ai-llm-service
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_ENDPOINT=http://langfuse:4318

OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.2
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=local
```
**Langfuse (self-hosted):**
```
LANGFUSE_PUBLIC_KEY=lf_pk_****
LANGFUSE_SECRET_KEY=lf_sk_****
DATABASE_URL=postgresql://langfuse:langfuse@postgres:5432/langfuse
```

### Commands to Run
```bash
docker compose pull
docker compose up -d
```
Startup order:
1. PostgreSQL
2. Langfuse services
3. Spring Boot application
---

### Expected outputs in Langfuse UI

- Traces grouped by `spring-ai-llm-service`
- Nested spans:
  - HTTP request span
  - Business logic span
  - Spring AI LLM invocation span
- Attributes visible per LLM span:
  - `llm.model`
  - `llm.prompt_tokens`
  - `llm.completion_tokens`
  - `llm.total_tokens`
  - `llm.latency_ms`
  - `error.type` (if applicable)

---

## 4) Data Model / Storage (Langfuse + PostgreSQL roles)

- **PostgreSQL** is the system of record for:
  - Traces and spans
  - Prompt/response payloads (subject to redaction)
  - Token usage and cost metadata

**Roles:**

- `langfuse_app`: read/write for ingestion and UI
- `langfuse_migration`: schema migrations only
- No application credentials are reused for observability storage.

Retention and indexing are managed by Langfuse migrations included in the stack.

---

## 5) Performance & Cost Controls (sampling, batching, redaction)

- **Sampling**
  - Parent-based trace ID ratio sampling (default 20%)
  - Always-on sampling for error spans

- **Batching**
  - OTEL batch span processor with bounded queue
  - Backpressure avoids application impact under load

- **Redaction**
  - Prompt/response content can be:
    - Fully disabled
    - Partially masked (regex-based)
  - Token counts and latency retained even when payloads are redacted

---

## 6) Security & Compliance Notes (PII, prompt/response handling)

- No data leaves the controlled environment.
- Langfuse is self-hosted and isolated on a private Docker network.
- PII handling strategies:
  - Pre-export redaction in the OTEL instrumentation layer
  - Environment-specific payload capture policies
- Secrets are injected via environment variables only; no secrets in images.

This setup supports internal compliance reviews and SOC-style audits.

---

## 7) Operational Runbook (monitoring, exporter failure, rollback)

**Monitoring**
- Monitor exporter error logs and queue saturation.
- Langfuse health endpoints exposed for liveness checks.

**Exporter failure**
- If OTLP endpoint is unavailable:
  - Spans are dropped after queue saturation.
  - Application traffic is not blocked.

**Rollback**
- Disable tracing by setting:
OTEL_TRACES_EXPORTER=none

- No application redeploy is required beyond configuration change.

---

## 8) Verification & Evidence Pack (trace screenshots, log lines)

The ZIP includes an evidence pack:

- **Logs**
- OTEL SDK initialization
- Successful OTLP exports
- **Screenshots**
- Langfuse trace view with nested Spring AI spans
- Token and latency breakdown
- **Checklist**
- Services running
- Traces visible
- Sampling verified
- Redaction confirmed

---

## 9) Changelog

### 1.0.0
- Initial release
- Spring AI instrumentation via OpenTelemetry
- Self-hosted Langfuse stack with PostgreSQL
- Docker Compose reproducibility

---

## 10) FAQ

**Q1: Does this support non-OpenAI Spring AI providers?**  
Yes. Instrumentation is provider-agnostic as long as Spring AI abstractions are used.

**Q2: Can prompts be completely excluded from storage?**  
Yes. Configuration supports metadata-only tracing.

**Q3: What is the overhead of tracing?**  
With sampling enabled, overhead is typically single-digit milliseconds per request.

**Q4: Can this run in Kubernetes instead of Docker Compose?**  
Yes. The same OTEL and Langfuse configuration applies; Compose is provided for local and CI reproducibility.

**Q5: How are costs calculated?**  
Langfuse derives cost from token counts and model pricing configuration.

**Q6: Is this compatible with existing APM tools?**  
Yes. OTEL spans can be dual-exported if required.

**Q7: What happens if Langfuse is down?**  
Tracing degrades gracefully; application traffic is unaffected.

**Q8: Can this be used in regulated environments?**  
Yes, with redaction enabled and controlled retention policies.
View [More runnable solutions here](https://exesolution.com)
