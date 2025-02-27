# Collecting Container Logs

<!-- TOC -->

- [Collecting Container Logs](#collecting-container-logs)
  - [Configuration](#configuration)
    - [Multiline log parsing](#multiline-log-parsing)
      - [Conditional multiline log parsing](#conditional-multiline-log-parsing)
    - [Log format](#log-format)
      - [json log format](#json-log-format)
      - [fields log format](#fields-log-format)
      - [json_merge log format](#json_merge-log-format)
      - [text log format](#text-log-format)
        - [Problem](#problem)
        - [Resolution](#resolution)
    - [Setting source name and other built-in metadata](#setting-source-name-and-other-built-in-metadata)
    - [Filtering](#filtering)
    - [Modifying log records](#modifying-log-records)
      - [Adding custom fields](#adding-custom-fields)
    - [Persistence](#persistence)
  - [Advanced Configuration](#advanced-configuration)
    - [Direct configuration](#direct-configuration)
    - [Disabling container logs](#disabling-container-logs)
    - [Using OTLP Source](#using-otlp-source)

<!-- /TOC -->

By default, log collection is enabled. This includes both container logs and systemd logs. This document covers container logs.

Container logs are read and parsed directly from the Node filesystem, where the kubelet writes them under the `/var/log/pods` directory.
They are then sent to a metadata enrichment service which takes care of adding Kubernetes metadata, custom processing, filtering, and
finally sending the data to Sumo Logic. Both the collection and the metadata enrichment are done by the OpenTelemetry Collector.

See the [Solution Overview diagram](README.md#log-collection) for a visualisation.

## Configuration

High level configuration for logs is located in [values.yaml][values] under the `sumologic.logs` key. Configuration specific to container
logs is located under the `sumologic.logs.container` key.

Configuration specific to the log collector DaemonSet can be found under the `otellogs` key.

Finally, configuration specific to the metadata enrichment StatefulSet can be found under the `metadata.logs` key.

### Multiline log parsing

By default, each line output by an application is treated as a separate log record. However, some applications can actually output logs
split into multiple lines - this is often the case for stack traces, for example. If we want such a multiline log to appear in Sumo Logic as
a single record, we need to tell the collector how to distinguish between lines which start a new record and ones which continue an existing
record.

Multiline log parsing can be configured using the `sumologic.logs.multiline` section in `user-values.yaml`.

```yaml
sumologic:
  logs:
    multiline:
      enabled: true
      first_line_regex: "^\\[?\\d{4}-\\d{1,2}-\\d{1,2}.\\d{2}:\\d{2}:\\d{2}"
```

where `first_line_regex` is a regular expression used to detect the first line of a multiline log.

This feature is enabled by default and the default regex will catch logs starting with a ISO8601 datetime. For example:

```text
2007-03-01T13:00:00Z this is the first line of a log record
  this is the second line
  and this is the third line
2007-03-01T13:00:01Z this is a new log record
```

This feature can rarely cause problems by merging together lines which are supposed to be separate. In that case, feel free to disable it.

#### Conditional multiline log parsing

Multiline log parsing can be also configured per specific conditions:

```yaml
sumologic:
  logs:
    multiline:
      enabled: true
      first_line_regex: "^\\[?\\d{4}-\\d{1,2}-\\d{1,2}.\\d{2}:\\d{2}:\\d{2}"
      additional:
        - first_line_regex: <regex 1>
          condition: <condition 1>
        - first_line_regex: <regex 2>
          condition: <condition 2>
        # ...
```

In that case `first_line_regex` of **first** matching condition is applied, and `sumologic.logs.multiline.first_line_regex` is used as
expression for logs which don't match any of the condition.

Conditions have to be valid [Open Telemetry Expression][expr].

The following variables may be used in the condition:

- `body` - body of a log
- `attributes["k8s.namespace.name"]`
- `attributes["k8s.pod.name"]`
- `attributes["k8s.container.name"]`
- `attributes["log.file.path"]` - log path on the node (`/var/log/pods/...`)
- `attributes["stream"]` - may be either `stdout` or `stderr`

Please consider the following example:

```yaml
sumologic:
  logs:
    multiline:
      enabled: true
      first_line_regex: "^\\[?\\d{4}-\\d{1,2}-\\d{1,2}.\\d{2}:\\d{2}:\\d{2}"
      additional:
        - first_line_regex: "^@@@@ First Line"
          condition: 'attributes["k8s.namespace.name"] == "foo"'
        - first_line_regex: "^--- First Line"
          condition: 'attributes["k8s.container.name"] matches "^bar-.*'
        - first_line_regex: "^Error"
          condition: 'attributes["stream"] == "stderr" and attributes["k8s.namespace.name"] != "lorem"'
```

It is going to:

- Use `^@@@@ First Line` expression for all logs from `foo` namespace
- Use `^--- First Line` expression for all remaingin logs from containers, which names start with `bar-`
- Use `^Error` expression for all remaining `stderr` logs which are not in `lorem` namespace
- Use `^\\[?\\d{4}-\\d{1,2}-\\d{1,2}.\\d{2}:\\d{2}:\\d{2}` expression for all remaining logs

**Note: Log which matches multiple conditions is processed only by the first one.**

[expr]: https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.90.1/pkg/stanza/docs/types/expression.md

### Log format

There are three log formats available: `fields`, `json_merge` and `text`. `fields` is the default.

You can change it by setting:

```yaml
sumologic:
  logs:
    container:
      format: fields
```

We're going to demonstrate the differences between them on two example log lines:

1. A plain text log

   ```text
   2007-03-01T13:00:00Z I am a log line
   ```

1. A JSON log

   ```json
   { "log_property": "value", "text": "I am a json log" }
   ```

#### `json` log format

`json` log format is an alias for `fields` log format.

See [`fields` log format](#fields-log-format)

#### `fields` log format

Logs formatted as `fields` are wrapped in a JSON object with additional properties, with the log body residing under the `log` key.

For example, log line 1 will show up in Sumo Logic as:

```javascript
{
  log: "2007-03-01T13:00:00Z I am a log line",
  stream: "stdout",
  timestamp: 1673627100045
}
```

If the log line contains json, as log line 2 does, it will be displayed as a nested object inside the `log` key:

```javascript
{
  log: {
    log_property: "value",
    text: "I am a json log"
  },
  stream: "stdout",
  timestamp: 1673627100045
}
```

#### `json_merge` log format

`json_merge` is identical to `fields` for non-JSON logs, but behaves differently for JSON logs. If the log is JSON, it gets merged into the
top-level object.

Log line 1 will show up the same way as it did for `fields`:

```javascript
{
  log: "2007-03-01T13:00:00Z I am a log line",
  stream: "stdout",
  timestamp: 1673627100045
}
```

However, the attributes from log line 2 will show up at the top level:

```javascript
{
  log: {
    log_property: "value",
    text: "I am a json log"
  },
  stream: "stdout",
  timestamp: 1673627100045
  log_property: "value",
  text: "I am a json log"
}
```

#### `text` log format

The `text` log format sends the log line as-is without any additional wrappers.

Log line 1 will therefore show up as plain text:

```text
2007-03-01T13:00:00Z I am a log line
```

Whereas log line 2 will be displayed as JSON:

```javascript
{
  log_property: "value",
  text: "I am a json log"
}
```

If you want to send metadata along wih log, you have to use reosurce level attributes, because record level attributes are going to be
removed before sending log to Sumo Logic. Please see [Mapping OpenTelemetry concepts to Sumo Logic][mapping] for more details.

##### Problem

If you changed log format to `text`, you need to know that multiline detection performed on the collection side is not respected anymore. As
we are sending logs as a wall of plain text, there is no way to inform Sumo Logic, which line belongs to which log. In such scenario,
multiline detection is performed on Sumo Logic side. By default it uses [Infer Boundaries][infer-boundaries]. You can review the source
configuration in [HTTP Source Settings][http-source].

**Note**: Your source name is going to be taken from `sumologic.collectorName` or `sumologic.clusterName` (`kubernetes` by default).

##### Resolution

In order to change multiline detection to [Boundary Regex][boundary-regex], for example to `\[?\d{4}-\d{1,2}-\d{1,2}.\d{2}:\d{2}:\d{2}.*`,
add the following configuration to your `user-values.yaml`:

```yaml
sumologic:
  collector:
    sources:
      logs:
        default:
          properties:
            ## Disable automatic multiline detection on collector side
            use_autoline_matching: false
            ## Set the following multiline detection regexes on collector side:
            ## - \{".* - in order to match json lines
            ## - \[?\d{4}-\d{1,2}-\d{1,2}.\d{2}:\d{2}:\d{2}.*
            ## Note: `\` is translated to `\\` and `"` to `\"` as we pass to terraform script
            manual_prefix_regexp: (\\{\".*|\\[?\\d{4}-\\d{1,2}-\\d{1,2}.\\d{2}:\\d{2}:\\d{2}.*)
```

**Note**: Double escape of `\` is needed, as well as escaping `"`, because value of `manual_prefix_regexp` is passed to terraform script.

**Note**: If you use `json` format along with `text` format, you need to add regex for `json` as well (`\\{\".*`)

**Note**: Details about `sumologic.collector.sources` configuration can be found [here][sumologic-terraform-provider]

[infer-boundaries]: https://help.sumologic.com/docs/send-data/reference-information/collect-multiline-logs#infer-boundaries
[http-source]: https://help.sumologic.com/docs/send-data/hosted-collectors/http-source
[boundary-regex]: https://help.sumologic.com/docs/send-data/reference-information/collect-multiline-logs#boundary-regex
[sumologic-terraform-provider]: ./terraform.md#sumo-logic-terraform-provider

### Setting source name and other built-in metadata

It's possible to customize the built-in Sumo Logic metadata (like [source name][source_name] for example) for container logs:

```yaml
sumologic:
  logs:
    container:
      ## Set the _sourceHost metadata field in Sumo Logic.
      sourceHost: ""
      ## Set the _sourceName metadata field in Sumo Logic.
      sourceName: "%{namespace}.%{pod}.%{container}"
      ## Set the _sourceCategory metadata field in Sumo Logic.
      sourceCategory: "%{namespace}/%{pod_name}"
      ## Set the prefix, for _sourceCategory metadata.
      sourceCategoryPrefix: "kubernetes/"
      ## Used to replace - with another character.
      sourceCategoryReplaceDash: "/"
```

As can be seen in the above example, these fields can contain templates of the form `%{field_name}`, where `field_name` is the name of a
resource attribute. Available resource attributes include the values of `sumologic.logs.fields`, which by default are:

- `cluster`
- `container`
- `daemonset`
- `deployment`
- `host`
- `namespace`
- `node`
- `pod`
- `service`
- `statefulset`

in addition to the following:

- `_collector`
- `pod_labels_*` where `*` is the Pod label name

### Filtering

Please see [the doc about filtering data](/docs/filtering.md).

### Modifying log records

To modify log records, use [OpenTelemetry processors][opentelemetry_processors]. Add them to
`sumologic.logs.container.otelcol.extraProcessors`.

Here are some examples.

To modify log body, use the [Transform processor][transform_processor_docs]:

```yaml
sumologic:
  logs:
    container:
      otelcol:
        extraProcessors:
          - transform/mask-card-numbers:
              log_statements:
                - context: log
                  statements:
                    - replace_pattern(body, "card=\\d+", "card=***")
```

To modify record attributes, use the [Attributes processor][attributes_processor_docs]:

```yaml
sumologic:
  logs:
    container:
      otelcol:
        extraProcessors:
          - attributes/delete-record-attribute:
              actions:
                - action: delete
                  key: unwanted.attribute
          # To rename old.attribute to new.attribute, first create new.attribute and then delete old.attribute.
          - attributes/rename-old-to-new:
              - action: insert
                key: new.attribute
                from_attribute: old.attribute
              - action: delete
                key: old.attribute
```

To modify resource attributes, use the [Resource processor][resource_processor_docs]:

```yaml
sumologic:
  logs:
    container:
      otelcol:
        extraProcessors:
          - resource/add-resource-attribute:
              attributes:
                - action: insert
                  key: environment
                  value: staging
          - resource/remove:
              attributes:
                - action: delete
                  key: redundant-attribute
```

#### Adding custom fields

To add a custom [field][sumo_fields] named `static-field` with value `hardcoded-value` to logs, use the following configuration:

```yaml
sumologic:
  logs:
    container:
      otelcol:
        extraProcessors:
          - resource/add-static-field:
              attributes:
                - action: insert
                  key: static-field
                  value: hardcoded-value
```

To add a custom field named `k8s_app` with a value that comes from e.g. the pod label `app.kubernetes.io/name`, use the following
configuration:

```yaml
sumologic:
  logs:
    container:
      otelcol:
        extraProcessors:
          - resource/add-k8s_app-field:
              attributes:
                - action: insert
                  key: k8s_app
                  from_attribute: pod_labels_app.kubernetes.io/name
```

> **Note** Make sure the field is [added in Sumo Logic][sumo_add_fields].

### Persistence

By default, the metadata enrichment service provisions and uses a Kubernetes PersistentVolume as an on-disk queue that guarantees durability
across Pod restarts and buffering in case of exporting problems.

This feature is enabled by default, but it only works if you have a correctly configured default `storageClass` in your cluster. Cloud
providers will do this for you when provisioning the cluster. The only alternative is disabling persistence altogether.

Persistence can be customized via the `metadata.logs.persistence` section:

```yaml
metadata:
  persistence:
    enabled: true
    # storageClass: ""
    accessMode: ReadWriteOnce
    size: 10Gi
    ## Add custom labels to all otelcol statefulset PVC (logs and metrics)
    pvcLabels: {}
```

> **Note** These settings affect persistence for metrics as well.

## Advanced Configuration

This section covers more advanced ways of configuring logging. Knowledge of OpenTelemetry Collector configuration format and concepts will
be required.

### Direct configuration

There are two ways of directly configuring OpenTelemetry Collector for both log collection and metadata enrichment. These are both advanced
features requiring a good understanding of this chart's architecture and OpenTelemetry Collector configuration.

The `metadata.logs.config.merge` and `otellogs.config.merge` keys can be used to provide configuration that will be merged with the Helm
Chart's default configuration. It should be noted that this field is not subject to normal backwards compatibility guarantees, the default
configuration can change even in minor versions while preserving the same end-to-end behaviour. Use of this field is discouraged - ideally
the necessary customizations should be able to be achieved without touching the otel configuration directly. Please open an issue if your
use case requires the use of this field.

The `metadata.logs.config.override` and `otellogs.config.override` keys can be used to provide configuration that will be completely replace
the default configuration. As above, care must be taken not to depend on implementation details that may change between minor releases of
this Chart.

See [Sumologic OpenTelemetry Collector configuration][configuration] for more information.

### Disabling container logs

Container logs are collected by default. This can be disabled by setting:

```yaml
sumologic:
  logs:
    container:
      enabled: false
```

### Using OTLP Source

OTLP source resolves some issues of `text` format, which affects HTTP source:

- Multiline logs are respected, Sumo will not split logs sent to an OTLP source
- Timestamp is set correctly to time set by Kubernetes, as OTLP contains a dedicated timestamp field.

[configuration]: https://github.com/SumoLogic/sumologic-otel-collector/blob/main/docs/configuration.md
[values]: /deploy/helm/sumologic/values.yaml
[source_name]: https://help.sumologic.com/docs/send-data/reference-information/metadata-naming-conventions/#Source_Name
[opentelemetry_processors]: https://opentelemetry.io/docs/collector/configuration/#processors
[attributes_processor_docs]:
  https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.69.0/processor/attributesprocessor/README.md
[resource_processor_docs]:
  https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.69.0/processor/resourceprocessor/README.md
[transform_processor_docs]:
  https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.69.0/processor/transformprocessor/README.md
[sumo_fields]: https://help.sumologic.com/docs/manage/fields/
[sumo_add_fields]: https://help.sumologic.com/docs/manage/fields/#add-field
[mapping]:
  https://help.sumologic.com/docs/send-data/opentelemetry-collector/data-source-configurations/additional-configurations-reference/#mapping-opentelemetry-concepts-to-sumo-logic
