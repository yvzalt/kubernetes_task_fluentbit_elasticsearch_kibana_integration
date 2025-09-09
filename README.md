# Kubernetes Logging Stack (Fluent Bit → Elasticsearch → Kibana)

This doc is a **guide** for building a Kubernetes logging stack using **Fluent Bit → Elasticsearch → Kibana**.  
It intentionally does **not** include production manifests for the stack—so you (or a junior) can design and reason about each piece.  
It **does** include several **log generator** pods to test your pipeline end-to-end.

---

## Goals

- Understand the core flow: **Container logs → Fluent Bit (DaemonSet) → Elasticsearch → Kibana**
- Make deliberate choices (paths, RBAC, buffers, ILM, security) rather than copy-pasting a Helm chart
- Verify the pipeline with realistic test logs (INFO/ERROR, JSON, multiline stack traces, burst traffic)

---

## High-level Steps (you implement the manifests)

1. **Setup Fluent Bit (DaemonSet)**
   - Tail `/var/log/containers/*.log`
   - Add Kubernetes metadata filter
   - Enable filesystem buffering for backpressure
   - Expose Prometheus metrics (optional)

2. **Test Fluent Bit (local)**
   - Deploy any log generator pod from `tests/`
   - Confirm records flow (INPUT → FILTER → OUTPUT) in Fluent Bit logs

3. **Configure Fluent Bit**
   - Parsers: `docker`/`cri`, JSON; enable a multiline parser (e.g., Java)
   - Optional: redact secrets; reduce noise with grep/modify filters
   - Output: retry policy, TLS/auth (prod)

4. **Setup Elasticsearch**
   - Single-node for dev
   - StatefulSet + PVCs (optional)
   - Decide index strategy: **data streams** (preferred) or daily indices (optional )
   - Add ILM: hot rollover + timed delete (dev example: keep 14 days)

5. **Integrate Fluent Bit → Elasticsearch**
   - Point FB `es` output at the ES service (cluster DNS)
   - Elasticsearch (index or data stream)
   - Choose index prefix (e.g., `kubernetes-logs`) or a data stream name
   - Verify with `_cat/indices`, `_search`, or `_data_stream`


6. **Test Integration**
   - Use the log generators (INFO/ERROR, JSON, multiline) and confirm ingestion
   - Check doc counts and fields (`kubernetes.*`) in ES

7. **Setup Kibana**
   - Deployment + Service (LB/Ingress or port-forward)
   - Configure Kib → ES hosts (and credentials if security enabled)

8. **Integrate Kibana ↔ Elasticsearch**
   - Create a **Data View** for `kubernetes-logs*` (or your data stream)
   - Time field: `@timestamp`

9. **Test ES ↔ Kibana**
   - Open **Discover**, see live entries, filter by namespace/pod

10. **Configure Kibana (usability)**
   - Saved search, simple dashboard (error rate, top noisy pods)
   - Optional: alerting rules for error spikes

11. **End-to-End Test**
   - Restart a pod; confirm timelines align in Kibana

12. **Check-Optional-Steps**
   - consider doing the optional steps
---

## Acceptance Criteria (suggested)

- **FB DaemonSet** ready on target nodes; tails expected log paths
- **Multiline** stacks appear as **single** events
- **ES** reachable; write path healthy; ILM applied (if you use indices)
- **Kibana** shows fresh logs; filters by `namespace/pod/container` work
- **Backpressure** handled (FB buffers while ES down, then recovers)

---

## Design Guardrails & Tips (optional for better system)

- **Paths**: containerd vs Docker paths differ—verify your nodes(we will be using containerd)
- **Security**: run FB as non-root, read-only root FS; use NetworkPolicies; TLS + creds
- **Buffers**: `storage.type filesystem` in FB for bursty workloads
- **ES Sizing**: start small—1 shard, 0–1 replica; JVM heap ~50% of container memory (≤ 32 GB)
- **Retention**: set ILM early; avoid runaway disk usage
- **Metrics**: FB retries, buffer size; ES indexing rate; alert on output retries > 0 for sustained periods

---

## What’s included here

- `tests/` **log generator** manifests (no image builds required):
  - `basic` (INFO/ERROR lines)
  - `json` (structured logs)
  - `multiline-java` (stack traces)
  - `burst` (high event rate)

> You can apply these immediately to validate your pipeline.

---

# Namespace for the stack and tests
kubectl create ns logging || true

# Apply any test generator(s)
kubectl apply -f tests/loggen-basic.yaml
kubectl apply -f tests/loggen-json.yaml
kubectl apply -f tests/loggen-multiline-java.yaml
# Optional burst:
kubectl apply -f tests/log-burst-job.yaml
