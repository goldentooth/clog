# OpenTelemetry and Tempo: Distributed Tracing for the Bramble

I have metrics (Prometheus) and logs (Loki). But there's a third pillar of observability I've been ignoring: **traces**.

When a request flows through multiple services, metrics tell me *something* is slow, and logs tell me *something* errored. But neither tells me the full story of what happened to *that specific request* as it bounced between services. That's what distributed tracing does.

Time to complete the observability trifecta.

## The OpenTelemetry Landscape

OpenTelemetry (OTel) is a CNCF project that standardizes how telemetry data (traces, metrics, logs) is collected and transmitted. Before OTel, every vendor had their own instrumentation libraries and protocols – Jaeger, Zipkin, Datadog, New Relic, all incompatible. OTel unifies this mess.

The key components:

- **OTLP (OpenTelemetry Protocol)**: The wire format for telemetry. Can run over gRPC (port 4317) or HTTP (port 4318).
- **SDKs**: Libraries you add to your code to generate telemetry
- **Collector**: A pipeline for receiving, processing, and exporting telemetry
- **Backends**: Where telemetry gets stored and queried (Jaeger, Tempo, Datadog, etc.)

The architecture follows a pipeline model:

```
┌────────────┐    ┌────────────┐    ┌────────────┐
│  Receivers │ -> │ Processors │ -> │  Exporters │
└────────────┘    └────────────┘    └────────────┘
```

Receivers ingest data (OTLP, Jaeger, Zipkin formats). Processors transform it (batching, filtering, sampling). Exporters send it to backends.

## The Missing Piece: Tempo

I already have Alloy running as my telemetry agent – it's collecting logs and shipping them to Loki. The beautiful thing about Alloy is it speaks OTel natively. It can be an OTLP receiver and exporter with a few lines of config.

But where do traces go? Loki stores logs, Prometheus stores metrics. I need a trace backend.

Enter **Grafana Tempo**. It's to traces what Loki is to logs:
- Accepts OTLP (and Jaeger, Zipkin) for ingestion
- Stores traces efficiently in a columnar format
- Queryable via TraceQL (similar to LogQL)
- Native Grafana integration

The full stack becomes LGTP: **L**oki, **G**rafana, **T**empo, **P**rometheus.

## Protocol Decision: gRPC vs HTTP

OTLP supports two transports:

**gRPC (port 4317)**:
- Binary protocol, more efficient
- HTTP/2 streaming
- ~30-40% better throughput

**HTTP (port 4318)**:
- JSON payloads, human-readable
- Works through any proxy
- Debug with curl

For a learning cluster, HTTP wins. I can test the endpoint with:

```bash
curl -X POST http://alloy:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans": [...]}'
```

Try doing *that* with gRPC. You'd need `grpcurl` or similar tooling.

## The Implementation

### Tempo Deployment

Created a new directory `gitops/infrastructure/tempo/` following the same pattern as Loki:

```yaml
# release.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: tempo
  namespace: flux-system
spec:
  chart:
    spec:
      chart: tempo
      version: '>=1.10.0 <2.0.0'
      sourceRef:
        kind: HelmRepository
        name: grafana
        namespace: flux-system
  values:
    tempo:
      storage:
        trace:
          backend: local
          local:
            path: /var/tempo/traces
      retention: 72h
      receivers:
        otlp:
          protocols:
            grpc:
              endpoint: "0.0.0.0:4317"
            http:
              endpoint: "0.0.0.0:4318"
    persistence:
      enabled: true
      storageClassName: local-path
      size: 10Gi
```

Using local filesystem storage because traces are typically shorter-lived than logs – 72 hours is plenty for debugging recent issues.

### Alloy OTLP Pipeline

Added a second pipeline to Alloy alongside the existing log collection:

```river
// OTLP Receiver - accepts traces from instrumented applications
otelcol.receiver.otlp "default" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }
  http {
    endpoint = "0.0.0.0:4318"
  }
  output {
    traces = [otelcol.processor.batch.default.input]
  }
}

// Batch Processor - groups traces before sending
otelcol.processor.batch "default" {
  send_batch_size = 8192
  timeout = "200ms"
  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}

// OTLP Exporter - sends to Tempo
otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "monitoring-tempo.monitoring.svc.cluster.local:4317"
    tls {
      insecure = true
    }
  }
}
```

The batch processor is important – instead of sending each span immediately, it groups them and sends batches every 200ms or 8KB, whichever comes first. This dramatically reduces network overhead.

Also had to expose the OTLP ports on Alloy's service so applications can reach it:

```yaml
alloy:
  extraPorts:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
      protocol: TCP
    - name: otlp-http
      port: 4318
      targetPort: 4318
      protocol: TCP
```

### Grafana Datasource with Correlations

The datasource config is where things get interesting:

```yaml
datasources:
  - name: Tempo
    type: tempo
    uid: tempo
    url: http://monitoring-tempo.monitoring.svc.cluster.local:3200
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki
        tags:
          - key: "namespace"
            value: "namespace"
          - key: "pod"
            value: "pod"
      tracesToMetrics:
        datasourceUid: prometheus
        tags:
          - key: "service.name"
            value: "service"
      serviceMap:
        datasourceUid: prometheus
```

This enables **correlations** between traces and other signals:
- `tracesToLogsV2`: Click a trace span, see logs from that pod during that time window
- `tracesToMetrics`: Jump from a trace to the service's Prometheus metrics
- `serviceMap`: Visualize service dependencies from trace data

The tag mappings tell Grafana how to translate between Tempo's attributes and Loki/Prometheus labels. When viewing a span with `namespace=monitoring`, clicking "logs" builds the query `{namespace="monitoring"}`.

### RED Metrics Generation

Enabled one of Tempo's killer features – automatic metrics generation from traces:

```yaml
metricsGenerator:
  enabled: true
  remoteWriteUrl: "http://monitoring-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090/api/v1/write"
  processors:
    - service-graphs
    - span-metrics
```

This creates RED metrics (Rate, Errors, Duration) from trace data and writes them to Prometheus. The beauty: your dashboard showing "p99 latency = 250ms" uses the exact same data as the trace you'll click to investigate *why* it's 250ms.

Also means you can sample traces (keep 10% to save storage) while still having 100% accurate metrics.

## The Bug: YAML Duplicate Keys

First deployment failed with:

```
yaml: unmarshal errors: line 229: mapping key "alloy" already defined
```

I had defined `alloy:` twice in the Helm values – once for the configMap content, once for extraPorts:

```yaml
values:
  alloy:
    configMap:
      content: |
        // ... river config ...

  # ... other stuff ...

  alloy:  # OOPS - duplicate key!
    extraPorts:
      - name: otlp-grpc
        ...
```

YAML parsers either error (good) or silently use only one of the values (bad). Fixed by merging extraPorts into the first `alloy:` block.

## The Final Architecture

```
                                    ┌─────────────┐
                                    │   Grafana   │ <- Query all three!
                                    └──────┬──────┘
                           ┌───────────────┼─────────────┐
                           v               v             v
                    ┌──────────┐    ┌──────────┐    ┌──────────┐
                    │Prometheus│    │   Loki   │    │  Tempo   │
                    │ (metrics)│    │  (logs)  │    │ (traces) │
                    └──────────┘    └────^─────┘    └────^─────┘
                                         │               │
                                    ┌────┴───────────────┴────┐
                                    │         Alloy           │
                                    │  (logs pipeline)        │
                                    │  (traces pipeline)      │
                                    └────────────^────────────┘
                                                 │ OTLP
                                    ┌────────────┴────────────┐
                                    │   Instrumented Apps     │
                                    └─────────────────────────┘
```

Applications send traces via OTLP to Alloy, which batches them and forwards to Tempo. Grafana can query Tempo with TraceQL and correlate with logs and metrics.

## Testing

Port-forward to Alloy and send a test trace:

```bash
kubectl port-forward -n monitoring svc/monitoring-alloy 4318:4318

curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "test-service"}}]},
      "scopeSpans": [{
        "spans": [{
          "traceId": "'$(openssl rand -hex 16)'",
          "spanId": "'$(openssl rand -hex 8)'",
          "name": "test-span",
          "kind": 1,
          "startTimeUnixNano": "'$(date +%s)000000000'",
          "endTimeUnixNano": "'$(( $(date +%s) + 1 ))000000000'"
        }]
      }]
    }]
  }'
```

Then in Grafana: Explore -> Tempo -> Search for `service.name = "test-service"`.
