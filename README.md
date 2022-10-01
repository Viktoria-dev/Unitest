# Unitest

Deployment Helm chart kube-prometheus-stack https://github.com/Viktoria-dev/Unitest/tree/main/prometheus/kube-prometheus-stack & prometheus-blackbox-exporter https://github.com/Viktoria-dev/Unitest/tree/main/prometheus/prometheus-blackbox-exporter

Github actions can deploy our two helm releases but we can do it in imperative way

We can use commands to installing our first kube-prometheus-stack release:


helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack -n unitest --values Unitest/prometheus/kube-prometheus-stack/values.yaml

And the next commands for installing blackbox-exporter release:

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install blackbox-exporter prometheus-community/prometheus-blackbox-exporter -n unitest --values Unitest/prometheus/prometheus-blackbox-exporter/values.yaml

seting up  prometheus-blackbox-exporter endpoint to google.com :


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
          
We also can seting up alerts to slack if it needed:

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
      

 We can also see our dashboards with port-forwarding k port-forward svc/prometheus-grafana -n unitest 3000:80
