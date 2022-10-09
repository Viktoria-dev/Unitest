# Unitest

Deployment Helm chart kube-prometheus-stack https://github.com/Viktoria-dev/Unitest/tree/main/prometheus/kube-prometheus-stack & prometheus-blackbox-exporter https://github.com/Viktoria-dev/Unitest/tree/main/prometheus/prometheus-blackbox-exporter

Github actions can deploy our two helm releases but we can do it in imperative way

We can use commands to installing our first kube-prometheus-stack release:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
```
helm install prometheus prometheus-community/kube-prometheus-stack -n unitest --values Unitest/prometheus/kube-prometheus-stack/values.yaml
```
And the next commands for installing blackbox-exporter release:
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install blackbox-exporter prometheus-community/prometheus-blackbox-exporter -n unitest --values Unitest/prometheus/prometheus-blackbox-exporter/values.yaml
```

seting up  prometheus-blackbox-exporter endpoint to google.com :



```
additionalScrapeConfigs: 
      - job_name: blackbox
        metrics_path: /probe
        params:
          module: [http_2xx]
        static_configs:
          # Add URLs as target parameter
          - targets:
            - https://www.google.com
        relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          # Important!     
          target_label: target
          # Ensure blackbox-exporter is reachable from Prometheus
        - target_label: __address__ 
          replacement: prometheus-blackbox-exporter.monitoring:9115 
```



We also can seting up alerts to slack if it needed:



```
receivers:
      - name: "null"
      - name: "slack"
        slack_configs:
          - api_url: "..."
            channel: "..."
            send_resolved: true
            icon_url: https://avatars3.githubusercontent.com/u/3380462
            title: |-
              [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }} for {{ .CommonLabels.job }}
              {{- if gt (len .CommonLabels) (len .GroupLabels) -}}
                {{" "}}(
                {{- with .CommonLabels.Remove .GroupLabels.Names }}
                  {{- range $index, $label := .SortedPairs -}}
                    {{ if $index }}, {{ end }}
                    {{- $label.Name }}="{{ $label.Value -}}"
                  {{- end }}
                {{- end -}}
                )
              {{- end }}
            text: >-
              {{ with index .Alerts 0 -}}
                :chart_with_upwards_trend: *<{{ .GeneratorURL }}|Graph>*
                {{- if .Annotations.runbook }}   :notebook: *<{{ .Annotations.runbook }}|Runbook>*{{ end }}
              {{ end }}
              *Alert details*:
              {{ range .Alerts -}}
                *Alert:* {{ .Annotations.title }}{{ if .Labels.severity }} - `{{ .Labels.severity }}`{{ end }}
              *Description:* {{ .Annotations.description }}
              *Details:*
                {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
                {{ end }}
              {{ end }}
    templates:
      - "/etc/alertmanager/config/*.tmpl"
```
 We can also see our dashboards with 
 ```
 kubctl port-forwarding k port-forward svc/prometheus-grafana -n unitest 3000:80 
 ```

Added dashboard which provides cluster admins with the ability to monitor nodes and identify workload bottlenecks. WITH ID: 10000

Changed password to "unitest123" but we can define it with `--set` flag
run local command  `password=unitest123`
`helm upgrade install prometheus prometheus-community/kube-prometheus-stack -n unitest --values Unitest/prometheus/kube-prometheus-stack/values.yaml --set $password`

Added alerts: CPU, memory, disk

```
additionalPrometheusRules: 
  - name: general.rules
    rules:
    - alert: HostOutOfMemory
      annotations:
        description: 'Node memory is filling up (< 80% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}'
        title: Host out of memory (instance {{ $labels.instance }})
      expr: cluster:namespace:pod_memory:active:kube_pod_container_resource_limits{namespace="unitest"} / 10000000000 >80
      for: 2m
      labels:
        severity: warning
    - alert: HostHighCpuLoad
      annotations:
        description: 'CPU load is > 80%\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}'
        title: Host high CPU load (instance {{ $labels.instance }})
      expr: cluster:namespace:pod_cpu:active:kube_pod_container_resource_limits{namespace="unitest"} >  80
      for: 2m
      labels:
        severity: warning
    - alert: Host-Out-Of-Disk-PVC
      annotations:
        description: 'Disk on PVC is almost full (< 20% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}'
        title: Host out of disk space (instance {{ $labels.instance }})
      expr: (max by (persistentvolumeclaim,namespace) (kubelet_volume_stats_used_bytes{namespace="unitest"})) / 1000000 > 80
      for: 2m
      labels:
        severity: warning
    - alert: HostOutOfDiskSpace
      annotations:
        description: 'Disk is almost full (< 80% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}'
        title: Host out of disk space (instance {{ $labels.instance }})
      expr: (node_filesystem_avail_bytes{fstype!="",job="node-exporter"} / node_filesystem_size_bytes{fstype!="",job="node-exporter"} * 100 < 20 and node_filesystem_readonly{fstype!="",job="node-exporter"} ==  0)
      for: 2m
      labels:
        severity: warning
    - alert: PVC-Storage-Class
      annotations:
        description: PVC-Storage-Class is almost full (< 80% left)
        title: PVC-Storage-Class out of disk space
      expr: (count(kube_persistentvolumeclaim_info {storageclass="efs-sc"} )*100)/120 > 80
      for: 2m
      labels:
        severity: warning
    - alert: PVC-Sandbox-efs
      annotations:
        description: PVC-Sandbox-efs is almost full (< 80% left)
        title: PVC-Sandbox-efs out of disk space
      expr: count(kube_persistentvolumeclaim_info{storageclass=~"sandbox-efs.*"}) * 100 / (count(kube_storageclass_info{provisioner="efs.csi.aws.com", storageclass=~"unitest-efs.*"})*120)  > 80
      for: 2m
      labels:
        severity: warning
```