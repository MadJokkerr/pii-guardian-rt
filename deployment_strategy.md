# Deployment Strategy — Project Guardian 2.0 (Real‑time PII Defense)

## Where it runs (recommended primary): API Gateway / Ingress Plugin
- **Placement:** NGINX/Kong Ingress Controller in Kubernetes.
- **Why here:** Intercepts *all* inbound/outbound JSON payloads before they reach any service. Lowest blast radius, zero code changes for apps.
- **Latency budget:** < 10 ms per request (PII scan + redact). Achieved via compiled regex, shallow JSON traversal, and short‑circuiting.
- **How:** 
  - Lua (Kong) or NGINX `njs` plugin that streams request/response bodies to a local **PII sidecar** via Unix domain socket.
  - Sidecar exposes `/scan` endpoint (FastAPI/uvicorn) using the exact logic from `detector_full_candidate_name.py` generalized for JSON bodies.
  - Redacted payload is forwarded downstream and/or logged; original PII is never persisted.

## Defense‑in‑Depth (secondary layers)
1. **DaemonSet (Logs & SIEM):** 
   - Fluent Bit → Sidecar `/scan` → SIEM (only redacted logs).
   - Blocks PII from entering centralized logging (cost & compliance win).

2. **Service Mesh (Envoy Filter):**
   - Optional Envoy WASM filter that mirrors payloads to the sidecar for scan/redact on sensitive services (User/Orders/Payments).

3. **Browser Extension (Internal Tools):**
   - Masks PII rendered in legacy internal tools without server changes (DOM scrubbing rules, auto‑mask by selectors + regex).

## Scale & Cost
- Horizontal pod autoscaling on the sidecar: CPU 60% target, min=2, max=N.
- Zero-copy streaming; chunked bodies to avoid OOM on big payloads.
- Cached DFA/compiled regex; warm pool of workers.
- Canary at ingress to validate false‑positives before global rollout.

## Observability & Guardrails
- Prometheus: `pii_scan_latency_ms`, `pii_detect_rate`, FP/FN samples.
- Circuit‑breaker: if scan > P95 budget or sidecar unhealthy → **pass‑through but redact known high‑risk fields** (phone/aadhar/upi/passport). 
- Shadow mode first, enforce mode later.

## Change Management
- Rulebook in ConfigMap (regex + field allow/deny lists).
- Feature flags to toggle categories (Standalone vs Combinatorial).
- Versioned policies, signed rule bundles.
