# Observability

Understanding running systems from their telemetry: monitoring, logging, tracing and alerting, and the tools for them.

## Map of the area

The diagram shows the notes in reading order; subareas are marked with (subarea).

```mermaid
flowchart LR
  n0["Observability overview"]
  n1["Monitoring"]
  n0 --> n1
  n2["Monitoring tools<br/>(subarea)"]
  n1 --> n2
  n3["Log management<br/>(subarea)"]
  n2 --> n3
```

## Notes in reading order

1. [Observability overview](observability-overview.md): signals, OpenTelemetry and the difference from monitoring.
2. [Monitoring](monitoring.md): categories, golden signals, alerting and tools.
3. [Monitoring tools](monitoring-tools/README.md): application monitoring, Nagios, New Relic, Splunk, AppDynamics, Instana.
4. [Log management](log-management/README.md): the log lifecycle, Elastic Stack, Graylog, Papertrail, Loki.

## Planned topics

- Prometheus
- Grafana
- OpenTelemetry in depth
- Distributed tracing (Jaeger, Tempo)
- Alerting and on-call (Alertmanager, PagerDuty)
- Zabbix
- Datadog
