Checklist with Acceptance Criteria

Setup Fluent Bit

DS runs on all (intended) nodes; paths mounted read-only.

Accept: desired=ready in DS status.

Test Fluent Bit

Deploy tests/log-generators/basic-pod.yaml.

Accept: FB logs show records flowing (INPUT→FILTER).

Configure Fluent Bit

Inputs: tail CRI logs; Parsers: docker/cri + multiline (Java).

Filters: kubernetes metadata; optional redaction & grep.

Buffering: filesystem on.

Accept: multiline events appear as single entries.

Setup Elasticsearch

Dev single-node with PVC.

Accept: GET / returns cluster info.

Integrate FB→ES

Output es with daily indices + ILM.

Accept: _cat/indices shows today’s index.

Test Integration

Use generators; count grows in ES.

Accept: _search returns recent docs with kube metadata.

Setup Kibana

LB/Ingress reachable.

Accept: Kibana status green/yellow.

Integrate Kibana↔ES

Kibana talks to ES service DNS.

Accept: Data view creation works.

Test ES↔Kibana

Data view kubernetes-logs*, time field @timestamp.

Accept: Discover shows live docs.

Configure Kibana

Saved search + starter dashboards.

Accept: Team filters by namespace/pod in <30s.

End-to-End Test

Restart a pod → verify timeline in Kibana.

Accept: Log path works: Node→FB→ES→Kib.

