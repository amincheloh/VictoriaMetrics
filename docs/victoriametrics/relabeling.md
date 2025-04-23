---
weight: 38
title: Relabeling cookbook
menu:
  docs:
    parent: 'victoriametrics'
    weight: 38
tags:
  - metrics
aliases:
- /relabeling.html
- /relabeling/index.html
- /relabeling/
---

The relabeling cookbook provides practical examples and patterns for transforming your metrics data as it flows through VictoriaMetrics, helping you control what gets collected and how it's labeled.

VictoriaMetrics and vmagent support Prometheus-style relabeling with [extra features](https://docs.victoriametrics.com/vmagent/#relabeling-enhancements) to enhance the functionality.

## Relabeling Stages

Relabeling in Prometheus happens in two main stages: during service discovery (`relabel_configs`), and during scraping (`metric_relabel_configs`).

Relabeling begins with `relabel_configs`, which are applied during the service discovery phase, before any scraping occurs.

These are applied during the service discovery phase, before VictoriaMetrics starts scraping any metrics. The goal here is to process and filter the list of targets that Prometheus discovers. You can add, remove, or modify target labels, or even drop targets altogether.

For example, you may want to scrape only the targets with the label `env=prod`:

```yaml
relabel_configs:
- source_labels: [env]
  regex: prod
  action: keep
```

This keeps only targets where the env label is `prod`, and drops the rest.

Once VictoriaMetrics has finished selecting the targets using `relabel_configs`, it starts scraping those endpoints. After scraping, you can apply `metric_relabel_configs`. This is the second stage, and it operates on **individual metrics**, not the targets. This means you can filter or modify the scraped time series before VictoriaMetrics stores them in its time series database.

## Relabeling Cheat Sheet

**Target-level relabeling** is applied during [service discovery](https://docs.victoriametrics.com/sd_configs/#prometheus-service-discovery) and affects the targets (which will be scraped), their labels and all the metrics scraped from them:

- [Drop targets](#how-to-drop-discovered-targets): Filter out unwanted targets from being scraped based on their labels
- [Configure scrape URLs](#how-to-modify-scrape-urls-in-targets): Change which URL is used to fetch metrics from each target
- [Add or update static labels](#how-to-add-labels-to-scrape-targets): Attach or update constant label values to all metrics from specific targets
- [Copy labels](#how-to-copy-labels-in-scrape-targets): Duplicate label values from one label to another
- [Modify instance/job labels](#how-to-modify-instance-and-job): Change the default `instance` and `job` labels for discovered targets
- [Extract label parts](#how-to-extract-label-parts): Parse and extract portions of label values into new labels
- [Remove prefixes](#how-to-remove-prefixes-from-target-label-names): Clean up label names by removing common prefixes
- [Remove labels](#how-to-remove-labels-from-targets): Delete specific labels from discovered targets

Note: All the target-level labels which are not prefixed with `__` are automatically added to all the metrics scraped from targets.

**Metric-level relabeling** is applied after metrics are scraped and affects the individual metrics:

- [Drop metrics](#how-to-drop-metrics-during-scrape): Filter out specific metrics to reduce cardinality and storage requirements
- [Rename metrics](#how-to-rename-scraped-metrics): Change metric names to follow naming conventions or standards
- [Add metric labels](#how-to-add-labels-to-scraped-metrics): Attach additional labels to scraped metrics for better querying
- [Change label values](#how-to-change-label-values-in-scraped-metrics): Modify existing label values to normalize or transform them
- [Remove metric labels](#how-to-remove-labels-from-scraped-metrics): Delete specific labels from scraped metrics
- [Remove labels with conditions](#how-to-remove-labels-from-metrics-subset): Delete labels only from metrics matching specific criteria

See also [relabeling docs at vmagent](https://docs.victoriametrics.com/vmagent/#relabeling).

## How to remove labels from metrics subset

You can remove certain labels from some metrics without affecting other labels by using the `if` parameter with `labeldrop` action. The `if` parameter is a [series selector](https://docs.victoriametrics.com/keyconcepts/#filtering) - it looks at the metric name and labels of each scraped time series.

For instance, this config below removes the `cpu` and `mode` labels, but only from the `node_cpu_seconds_total` metric where `mode="idle"` ([Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+action%3A+labeldrop%0A++if%3A+%27node_cpu_seconds_total%7Bmode%3D%22idle%22%7D%27%0A++regex%3A+%22cpu%7Cmode%22&labels=node_cpu_seconds_total%7Bmode%3D%22idle%22%2Cnode%3D%22A%22%7D)):
  ```yaml
  metric_relabel_configs:
    - action: labeldrop
      if: 'node_cpu_seconds_total{mode="idle"}'
      regex: "cpu|mode"
  ```

## How to rename scraped metrics

The metric name is actually the value of a special label called `__name__` (see [Key Concepts](https://docs.victoriametrics.com/keyconcepts/#labels)). So renaming a metric is performed in the same way as changing a label value. Let's take some examples:

- Rename `node_cpu_seconds_total` to `vm_node_cpu_seconds_total` across all the scraped metrics ([Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+if%3A+%27node_cpu_seconds_total%27%0A++replacement%3A+vm_node_cpu_seconds_total%0A++target_label%3A+__name__&labels=node_cpu_seconds_total%7Bcpu%3D%220%22%2C+mode%3D%22idle%22%7D)):
  ```yaml
  metric_relabel_configs:
  - if: 'node_cpu_seconds_total'
    replacement: vm_node_cpu_seconds_total
    target_label: __name__
  ```

- Rename all metrics starting with `http_` to start with `web_` instead (e.g. `http_requests_total` → `web_requests_total`, [Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+source_labels%3A+%5B__name__%5D%0A++regex%3A+%27http_%28.*%29%27%0A++replacement%3A+web_%241%0A++target_label%3A+__name__&labels=http_response_time_seconds%7Bmethod%3D%22GET%22%7D)):
  ```yaml
  metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'http_(.*)'
    replacement: web_$1
    target_label: __name__
  ```

- Replace all dashes (`-`) in metric names with underscores (`_`) (e.g. `nginx-ingress-latency` → `nginx_ingress_latency`, [Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+source_labels%3A+%5B__name__%5D%0A++action%3A+replace_all%0A++regex%3A+%27-%27%0A++replacement%3A+%27_%27%0A++target_label%3A+__name__&labels=nginx-ingress-latency%7Bhost%3D%22example.com%22%7D)):
  ```yaml
  metric_relabel_configs:
  - source_labels: [__name__]
    action: replace_all
    regex: '-'
    replacement: '_'
    target_label: __name__
  ```

## How to add labels to scraped metrics

You can add custom labels to scraped metrics using `target_label` to set the label name and the `replacement` field to set the label value. For example:

- Add a `region="us-east-1"` label to all scraped metrics ([Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+target_label%3A+region%0A++replacement%3A+us-east-1&labels=node_memory_MemAvailable_bytes%7Binstance%3D%22server-01%3A9100%22%7D)):
  ```yaml
  metric_relabel_configs:
  - target_label: region
    replacement: us-east-1
  ```

- Add a `team="platform"` label only for metrics from jobs that match `web-.*` and are not in the staging environment ([Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+if%3A+%27%7Bjob%3D%7E%22web-.*%22%2C+environment%21%3D%22staging%22%7D%27%0A++target_label%3A+team%0A++replacement%3A+platform&labels=http_requests_total%7Bjob%3D%22web-api%22%2C+environment%3D%22prod%22%7D)):
  ```yaml
  metric_relabel_configs:
  - if: '{job=~"web-.*", environment!="staging"}'
    target_label: team
    replacement: platform
  ```

## How to change label values in scraped metrics

To change the label values of scraped metrics, we use the following fields:
- `target_label`: the label we want to modify (if it exists) or create,
- `source_labels`: the label(s) whose values are used to compute the new value for `target_label`,
- `replacement`: the value that will be computed and assigned to the `target_label`.

Below are a few illustrations:

- Add prod_ prefix to all values of the job label across all scraped metrics ([Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+source_labels%3A+%5Bjob%5D%0A++target_label%3A+job%0A++replacement%3A+prod_%241&labels=node_memory_Active_bytes%7Bjob%3D%22node-exporter%22%2C+instance%3D%2210.0.0.1%3A9100%22%7D)):
  ```yaml
  metric_relabel_configs:
  - source_labels: [job]
    target_label: job
    replacement: prod_$1
  ```

- Add `prod_` prefix to `job` label values only for metrics matching `{job=~"api-service-.*",env!="dev"}` ([Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+if%3A+%27%7Bjob%3D%7E%22api-service-.*%22%2Cenv%21%3D%22dev%22%7D%27%0A++source_labels%3A+%5Bjob%5D%0A++target_label%3A+job%0A++replacement%3A+prod_%241&labels=http_requests_total%7Bjob%3D%22api-service-orders%22%2C+env%3D%22staging%22%2C+instance%3D%2210.0.0.5%3A8080%22%7D)):
  ```yaml
  metric_relabel_configs:
  - if: '{job=~"api-service-.*",env!="dev"}'
    source_labels: [job]
    target_label: job
    replacement: prod_$1
  ```

## How to remove labels from scraped metrics

Removing labels from scraped metrics is a good idea to avoid [high cardinality](https://docs.victoriametrics.com/faq/#what-is-high-cardinality) and [high churn rate](https://docs.victoriametrics.com/faq/#what-is-high-churn-rate) issues.

This can be done with either of the following actions:
- `action: labeldrop`: drops labels with names matching the given `regex` option
- `action: labelkeep`: drops labels with names not matching the given `regex` option

Let's see this in action:

- Remove labels with names starting with the `kubernetes_` prefix from all scraped metrics ([Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+action%3A+labeldrop%0A++regex%3A+%22kubernetes_.*%22&labels=container_cpu_usage_seconds_total%7Bcontainer%3D%22app%22%2C+kubernetes_namespace%3D%22default%22%2C+kubernetes_pod_name%3D%22app-123%22%7D)): 
  ```yaml
  metric_relabel_configs:
  - action: labeldrop
    regex: "kubernetes_.*"
  ```

The `regex` option must match the whole label name from start to end, not just a part of it.

Note that:

- Labels that start with `__` are removed automatically after relabeling, so you don't need to drop them with relabeling rules.

## How to drop metrics during scrape

All examples above work at the label level: adding, dropping, or changing label values of scraped metrics. You can also drop entire metrics. This is especially beneficial for metrics that result in [high cardinality](https://docs.victoriametrics.com/faq/#what-is-high-cardinality) or [high churn rate](https://docs.victoriametrics.com/faq/#what-is-high-churn-rate).

Instead of `labeldrop` or `labelkeep` actions, we use `drop` or `keep` actions in the `metric_relabel_configs` section:

- `action: drop`: drops all metrics that match the `if` [series selector](https://docs.victoriametrics.com/keyconcepts/#filtering)
- `action: keep`: drops all metrics that don't match the `if` [series selector](https://docs.victoriametrics.com/keyconcepts/#filtering)

For example, the following config drops all metrics with names starting with `container_` ([Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+if%3A+%27%7B__name__%3D%7E%22container_.*%22%7D%27%0A++action%3A+drop&labels=container_memory_usage_bytes%7Bcontainer%3D%22nginx%22%2C+pod%3D%22web-1%22%7D)):

```yaml
metric_relabel_configs:
- if: '{__name__=~"container_.*"}'
  action: drop
```

Note that the relabeling config is specified under the `metric_relabel_configs` section instead of `relabel_configs` section. They serve different purposes:

- The `scrape_configs[].relabel_configs` apply before scraping, modifying or filtering targets. Any changes here affect all metrics from that target.
- The `scrape_configs[].metric_relabel_configs` apply after scraping, modifying or filtering individual metrics.

## How to remove labels from targets

To remove some labels from targets discovered by the scrape job, use either:
- `action: labeldrop`: drops labels with names matching the given `regex` option
- `action: labelkeep`: drops labels with names not matching the given `regex` option

For example:

```yaml
scrape_configs:
- job_name: k8s
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - action: labelmap
    regex: "__meta_kubernetes_pod_label_(.+)"
    replacement: "pod_label_$1"
  - action: labeldrop
    regex: "pod_label_team_.*"
```

[Try the above config here.](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+action%3A+labelmap%0A++regex%3A+%22__meta_kubernetes_pod_label_%28.%2B%29%22%0A++replacement%3A+%22pod_label_%241%22%0A-+action%3A+labeldrop%0A++regex%3A+%22pod_label_team_.*%22&labels=container_memory_usage_bytes%7Bpod%3D%22nginx-abc123%22%2C+container%3D%22nginx%22%2C+namespace%3D%22default%22%2C+__meta_kubernetes_pod_label_app_kubernetes_io_name%3D%22nginx%22%2C+__meta_kubernetes_pod_label_team_backend%3D%22infra%22%2C+__meta_kubernetes_pod_label_team_frontend%3D%22dashboard%22%2C+__meta_kubernetes_pod_label_env%3D%22production%22%7D)

The job above will:

1. discover pods in [Kubernetes](https://docs.victoriametrics.com/sd_configs/#kubernetes_sd_configs)
2. extract pod-level labels (e.g. `app.kubernetes.io/name` and `team`)
3. prefix them with `pod_label_` and add them as labels to all scraped metrics
4. drop all labels starting with `pod_label_team_`
5. drop all labels starting with `__` (this is done by default by VictoriaMetrics)

Note that:

- Labels that start with `__` are removed automatically after relabeling, so you don't need to drop them with relabeling rules.
- Do not remove `instance` and `job` labels, since this may result in duplicate scrape targets with identical sets of labels.
- The `regex` option must match the whole label name from start to end, not just a part of it.

## How to remove labels from a subset of targets

To remove some target-labels from a subset of discovered targets, use the `if` [series selector](https://docs.victoriametrics.com/keyconcepts/#filtering) with `action: labeldrop` or `action: labelkeep` relabeling rule.

As an illustration:

- The job below discovers Kubernetes pods, extracts pod-level labels, adds them to all metrics with prefix `pod_label_`, and removes any labels starting with `pod_label_internal_` for **only targets matching `{__address__=~"pod123.+"}` selector**:

```yaml
scrape_configs:
- job_name: k8s
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - action: labelmap
    regex: "__meta_kubernetes_pod_label_(.+)"
    replacement: "pod_label_$1"
  - action: labeldrop
    if: '{__address__=~"pod123.+"}'
    regex: "pod_label_internal_.*"
```

[Try the above config here.](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+action%3A+labelmap%0A++regex%3A+%22__meta_kubernetes_pod_label_%28.%2B%29%22%0A++replacement%3A+%22pod_label_%241%22%0A-+action%3A+labeldrop%0A++if%3A+%27%7B__address__%3D%7E%22pod123.%2B%22%7D%27%0A++regex%3A+%22pod_label_internal_.*%22&labels=container_cpu_usage_seconds_total%7B__address__%3D%22pod123-api-0.default.svc%3A8080%22%2C__meta_kubernetes_pod_label_app%3D%22api%22%2C__meta_kubernetes_pod_label_env%3D%22staging%22%2C__meta_kubernetes_pod_label_internal_cost_center%3D%22devops%22%2C__meta_kubernetes_pod_label_internal_sensitive%3D%22true%22%7D)

## How to remove prefixes from target label names

You can modify target-labels including removing prefixes with the `action: labelmap` option.

For example, [Kubernetes service discovery](https://docs.victoriametrics.com/sd_configs/#kubernetes_sd_configs) automatically adds special `__meta_kubernetes_pod_label_<labelname>` labels for each pod-level label. 

All labels with the prefix `__` will be dropped automatically. To extract and keep only the `<labelname>` part of this special label, you can use `action: labelmap` combined with `regex` and `replacement` options:

```yaml
scrape_configs:
- job_name: k8s
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - action: labelmap
    regex: "__meta_kubernetes_pod_label_(.+)"
    replacement: "$1"
```

[Try the above config here.](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+action%3A+labelmap%0A++regex%3A+%22__meta_kubernetes_pod_label_%28.%2B%29%22%0A++replacement%3A+%22%241%22&labels=container_cpu_usage_seconds_total%7B__address__%3D%2210.42.3.57%3A8080%22%2C+container%3D%22nginx%22%2C+pod%3D%22nginx-prod-5d9f%22%2C+namespace%3D%22default%22%2C+__meta_kubernetes_pod_label_app%3D%22nginx%22%2C+__meta_kubernetes_pod_label_env%3D%22production%22%2C+__meta_kubernetes_pod_label_team%3D%22devops%22%7D)

The regex contains a capture group `(.+)`. This capture group can be referenced inside the `replacement` option with the `$N` syntax, such as `$1` for the first capture group.

This config will create a new label with the name extracted from the regex capture group `(.+)` for all metrics scraped from the discovered pods.

Note that:

- The `regex` option must match the whole label name from start to end, not just a part of it.

## How to extract label parts

Relabeling allows extracting parts from label values and storing them into arbitrary labels. This is performed with:

- `source_labels`: the label(s) whose values are used to compute the new value for `target_label`,
- `target_label`: the label we want to modify or create,
- `replacement`: the value that will be computed and assigned to the `target_label`,
- `regex`: the regular expression to be applied to the value of `source_labels`.

Let's take this case:

```yaml
scrape_configs:
- job_name: k8s
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_container_name]
    regex: "[^/]+/(.+)"
    replacement: "team_$1"
    target_label: owner_team
```

[Try the above config here](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+source_labels%3A+%5B__meta_kubernetes_pod_container_name%5D%0A++regex%3A+%22%5B%5E%2F%5D%2B%2F%28.%2B%29%22%0A++replacement%3A+%22team_%241%22%0A++target_label%3A+owner_team&labels=container_cpu_usage_seconds_total%7B__address__%3D%2210.42.3.99%3A8080%22%2Cpod%3D%22orders-backend-5489f%22%2Cnamespace%3D%22production%22%2Ccontainer%3D%22backend%22%2C__meta_kubernetes_pod_container_name%3D%22app%2Fbackend%22%7D)

The job above discovers pod targets in [Kubernetes](https://docs.victoriametrics.com/sd_configs/#kubernetes_sd_configs), and performs these actions:

1. Extracts the value of `__meta_kubernetes_pod_container_name` label (e.g. `foo/bar`), 
2. Matches it against the regex `[^/]+/(.+)`, 
3. Computes the new value as `team_$1` with `$1` capture from regex `(.+)`, 
4. Stores the result in the `owner_team` label.

Note that:

- The `regex` option must match the whole label value from start to end, not just a part of it.
- If `source_labels` contains multiple labels, their values are joined with a `;` separator (customized by the `separator` option) before being matched against the `regex`.

## How to modify instance and job

`instance` and `job` labels are automatically added by single-node VictoriaMetrics and [vmagent](https://docs.victoriametrics.com/vmagent/) for each discovered target.

- The `job` label is set to the `job_name` value specified in the corresponding `scrape_config`.
- The `instance` label is set to the `host:port` part of the `__address__` label value after target-level relabeling. The `__address__` label value depends on the type of [service discovery](https://docs.victoriametrics.com/sd_configs/#supported-service-discovery-configs) and [can be overridden](https://docs.victoriametrics.com/sd_configs/#scrape_configs) during relabeling.

Modifying `instance` and `job` labels works like other target-labels by using `target_label` and `replacement` options:

```yaml
scrape_configs:
- job_name: k8s
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - target_label: job
    replacement: kubernetes_pod_metrics
```

[Try the above config here](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+target_label%3A+job%0A++replacement%3A+kubernetes_pod_metrics&labels=container_memory_usage_bytes%7B__address__%3D%2210.42.3.99%3A8080%22%2C+container%3D%22checkout%22%2C+pod%3D%22checkout-api-5d9f%22%2C+namespace%3D%22production%22%2C+job%3D%22k8s%22%2C+instance%3D%2210.42.3.99%3A8080%22%7D+65432100)

## How to modify scrape URLs in targets

URLs for scrape targets are composed of the following parts:

- Scheme (e.g. `http`, `https`) is available during target relabeling in a special label - `__scheme__`. By default, it's set to `http` but can be overridden either by specifying the `scheme` option at [scrape_config](https://docs.victoriametrics.com/sd_configs/#scrape_configs) level or by updating the `__scheme__` label during relabeling.
- Host and port (e.g. `host12:3456`) is available during target relabeling in a special label - `__address__`. Its value depends on the [service discovery type](https://docs.victoriametrics.com/sd_configs/#supported-service-discovery-configs). Sometimes this value needs to be modified. In this case, just update the `__address__` label during relabeling to the needed value. 
  - The port part is optional. If it is missing, it's automatically set depending on the scheme (`80` for `http` or `443` for `https`). The `host:port` part from the final `__address__` label is automatically set to the `instance` label. The `__address__` label can contain the full scrape URL (e.g. `http://host:port/metrics/path?query_args`). In this case the `__scheme__` and `__metrics_path__` labels are ignored.
- URL path (e.g. `/metrics`) is available during target relabeling in a special label - `__metrics_path__`. By default, it's set to `/metrics` and can be overridden either by specifying the `metrics_path` option at [scrape_config](https://docs.victoriametrics.com/sd_configs/#scrape_configs) level or by updating the `__metrics_path__` label during relabeling.
- Query args (e.g. `?foo=bar&baz=xyz`) are available during target relabeling in special labels with the `__param_` prefix. 
  - Take `?foo=bar&baz=xyz` for example. There will be two special labels: `__param_foo="bar"` and `__param_baz="xyz"`. The query args can be specified either via the `params` section at [scrape_config](https://docs.victoriametrics.com/sd_configs/#scrape_configs) or by updating/setting the corresponding `__param_*` labels during relabeling.

The resulting scrape URL looks like the following:

```go
<__scheme__> + "://" + <__address__> + <__metrics_path__> + <"?" + query_args_from_param_labels>
```

Given the scrape URL construction rules above, the following config discovers pod targets in [Kubernetes](https://docs.victoriametrics.com/sd_configs/#kubernetes_sd_configs) and constructs a per-target scrape URL as `https://<pod_name>/metrics/container?name=<container_name>`:

```yaml
scrape_configs:
- job_name: k8s
  kubernetes_sd_configs:
  - role: pod
  metrics_path: /metrics/container
  relabel_configs:
  - target_label: __scheme__
    replacement: https
  - source_labels: [__meta_kubernetes_pod_name]
    target_label: __address__
  - source_labels: [__meta_kubernetes_pod_container_name]
    target_label: __param_name
```

[Try the above config here](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+target_label%3A+__scheme__%0A++replacement%3A+https%0A-+source_labels%3A+%5B__meta_kubernetes_pod_name%5D%0A++target_label%3A+__address__%0A-+source_labels%3A+%5B__meta_kubernetes_pod_container_name%5D%0A++target_label%3A+__param_name&labels=container_cpu_usage_seconds_total%7B__meta_kubernetes_pod_name%3D%22checkout-api-58c9d%22%2C+__meta_kubernetes_pod_container_name%3D%22app%22%2C+__meta_kubernetes_namespace%3D%22production%22%2C+__meta_kubernetes_pod_ip%3D%2210.42.6.25%22%2C+__address__%3D%2210.42.6.25%3A9100%22%2C+job%3D%22k8s%22%2C+instance%3D%2210.42.6.25%3A9100%22%7D)

## How to copy labels in scrape targets

Labels can be copied using the following options:

- `source_labels`: specifies which labels to copy from
- `target_label`: specifies the destination label to receive the value

The following config copies the `__meta_kubernetes_pod_name` label to the `pod` label for all discovered pods in [Kubernetes](https://docs.victoriametrics.com/sd_configs/#kubernetes_sd_configs):

```yaml
scrape_configs:
- job_name: k8s
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_name]
    target_label: pod
```

[Try the above config here](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+source_labels%3A+%5B__meta_kubernetes_pod_name%5D%0A++target_label%3A+pod&labels=container_cpu_usage_seconds_total%7B__meta_kubernetes_pod_name%3D%22nginx-deployment-65f7c58c5b-bxjzs%22%2Cnamespace%3D%22default%22%2Ccontainer%3D%22nginx%22%2Cpod_name%3D%22nginx-deployment-65f7c58c5b-bxjzs%22%7D)

If `source_labels` contains multiple labels, their values are joined with a `;` delimiter by default. Use the `separator` option to change this delimiter.

For example, this config combines pod name and container port into the `host_port` label for all discovered pod targets in [Kubernetes](https://docs.victoriametrics.com/sd_configs/#kubernetes_sd_configs):

```yaml
scrape_configs:
- job_name: k8s
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_name, __meta_kubernetes_pod_container_port_number]
    separator: ":"
    target_label: host_port
```

[Try the above config here](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+source_labels%3A+%5B__meta_kubernetes_pod_name%2C+__meta_kubernetes_pod_container_port_number%5D%0A++separator%3A+%22%3A%22%0A++target_label%3A+host_port&labels=container_network_receive_bytes_total%7B__meta_kubernetes_pod_name%3D%22api-server-746d95f76f-7r2hp%22%2C__meta_kubernetes_pod_container_port_number%3D%228080%22%2Cnamespace%3D%22backend%22%2Ccontainer%3D%22api%22%2Cinterface%3D%22eth0%22%7D)

## How to add labels to scrape targets

To add or update labels on scrape targets during discovery, use these options:

- `target_label`: specifies the label name to add or update
- `replacement`: specifies the value to assign to this label

For example, this config adds a `environment="production"` label to all discovered pods in [Kubernetes](https://docs.victoriametrics.com/sd_configs/#kubernetes_sd_configs):

```yaml
scrape_configs:
- job_name: k8s
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - target_label: "environment"
    replacement: "production"
```

[Try the above config here](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+target_label%3A+%22environment%22%0A++replacement%3A+%22production%22&labels=container_memory_usage_bytes%7Bcontainer%3D%22redis%22%2C+namespace%3D%22cache%22%2C+pod%3D%22redis-cache-9df49c5b9-hxz6m%22%7D)

If there is a conflict between target labels and metrics exported by the target (scrape-time labels), the `exported_` prefix is added to scrape-time labels.

To keep the scrape-time labels unchanged and let them override target labels, specify `honor_labels: true` in the scrape config. This gives priority to the labels from the scraped metrics.

For example, this config adds a `environment="production"` label to all discovered pods, but if any pod already exports a `environment` label, that value will override the target label:

```yaml
scrape_configs:
- job_name: k8s
  kubernetes_sd_configs:
  - role: pod
  honor_labels: true # <--
  relabel_configs:
  - target_label: "environment"
    replacement: "production"
```

See also [useful tips for target relabeling](#useful-tips-for-target-relabeling).

## How to drop discovered targets

To drop a particular discovered target, use the following options:

- `action: drop`: drops scrape targets with labels matching the `if` [series selector](https://docs.victoriametrics.com/keyconcepts/#filtering)
- `action: keep`: keeps scrape targets with labels matching the `if` series selector, while dropping all other targets

Here are examples of these options:

- This config discovers pods in [Kubernetes](https://docs.victoriametrics.com/sd_configs/#kubernetes_sd_configs) and drops all pods with names starting with the `test-` prefix ([Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+if%3A+%27%7B__meta_kubernetes_pod_name%3D%7E%22test-.*%22%7D%27%0A++action%3A+drop&labels=http_requests_total%7B__meta_kubernetes_pod_name%3D%22test-payment-7cbd8d77b6-4l5xv%22%2Cnamespace%3D%22qa%22%2Capp%3D%22payment-service%22%7D)):
  ```yaml
  scrape_configs:
  - job_name: prod_pods_only
    kubernetes_sd_configs:
    - role: pod
    relabel_configs:
    - if: '{__meta_kubernetes_pod_name=~"test-.*"}'
      action: drop
  ```

- This config keeps only pods with names starting with the `backend-` prefix ([Try it](https://play.victoriametrics.com/select/0/prometheus/graph/#/relabeling?config=-+if%3A+%27%7B__meta_kubernetes_pod_name%3D%7E%22backend-.*%22%7D%27%0A++action%3A+keep&labels=container_memory_usage_bytes%7B__meta_kubernetes_pod_name%3D%22frontend-auth-5cbdbb7ff8-qf82n%22%2Cnamespace%3D%22prod%22%2Ccontainer%3D%22auth%22%7D)):
  ```yaml
  scrape_configs:
  - job_name: backend_pods
    kubernetes_sd_configs:
    - role: pod
    relabel_configs:
    - if: '{__meta_kubernetes_pod_name=~"backend-.*"}'
      action: keep
  ```

See also [useful tips for target relabeling](#useful-tips-for-target-relabeling).

## Useful tips for target relabeling

- Target relabeling can be debugged by clicking the `debug` link for a target on the `http://vmagent:8429/target` or `http://vmagent:8429/service-discovery` pages. See [Relabel Debug - vmagent](https://docs.victoriametrics.com/vmagent/#relabel-debug).
- Special labels with the `__` prefix are automatically added when discovering targets and removed after relabeling:
  - Meta-labels starting with the `__meta_` prefix. The specific sets of labels for each supported service discovery option are listed in [Prometheus Service Discovery](https://docs.victoriametrics.com/sd_configs/#prometheus-service-discovery).
  - Additional labels with the `__` prefix other than `__meta_` labels, such as [`__scheme__` or `__address__`](#how-to-modify-scrape-urls-in-targets).
  It is common practice to store temporary labels with names starting with `__` during target relabeling.
- All target-level labels are automatically added to all metrics scraped from targets.
- The list of discovered scrape targets with all discovered meta-labels is available on the `http://vmagent:8429/service-discovery` page for `vmagent` and on the `http://victoriametrics:8428/service-discovery` page for single-node VictoriaMetrics.
- The list of active targets with the final set of target-labels after relabeling is available on the `http://vmagent:8429/targets` page for `vmagent` and on the `http://victoriametrics:8428/targets` page for single-node VictoriaMetrics.

## Useful tips for metric relabeling

- Metric relabeling can be debugged on the `http://vmagent:8429/metric-relabel-debug` page. See [these docs](https://docs.victoriametrics.com/vmagent/#relabel-debug).
- All labels that start with the `__` prefix are automatically removed from metrics after relabeling. It is common practice to store temporary labels with names starting with `__` during metrics relabeling.
- All target-level labels are automatically added to all metrics scraped from targets, making them available during metrics relabeling.
- If too many labels are removed, different metrics might look the same — this can lead to duplicate time series with conflicting values, which is usually a problem.