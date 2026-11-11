# Log management

Collecting, storing and searching logs at scale, and the main log platforms.

## Map

The diagram shows the notes in reading order; subareas are marked with (subarea).

```mermaid
flowchart LR
  n0["Log management"]
  n1["Elastic Stack"]
  n0 --> n1
  n2["Graylog"]
  n1 --> n2
  n3["Papertrail"]
  n2 --> n3
  n4["Loki"]
  n3 --> n4
```

## Notes in reading order

1. [Log management](log-management-overview.md): lifecycle, structured logging and syslog.
2. [Elastic Stack](elastic-stack.md): Elasticsearch, Logstash, Kibana and Beats.
3. [Graylog](graylog.md): inputs, streams, pipelines and index sets.
4. [Papertrail](papertrail.md): hosted live tail and search.
5. [Loki](loki.md): label-indexed logs and LogQL.

## Planned topics

- Fluent Bit and Fluentd
- OpenSearch
- Vector
- Log retention and compliance
