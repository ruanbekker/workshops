
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo prometheus-community/kube-prometheus-stack --versions

NAME                                      	CHART VERSION	APP VERSION	DESCRIPTION
prometheus-community/kube-prometheus-stack	40.5.0       	0.59.2     	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	40.4.0       	0.59.2     	kube-prometheus-stack collects Kubernetes manif...
```

```
helm show values prometheus-community/kube-prometheus-stack  --version 40.5.0 > grafana-values.yaml
```

```
helm upgrade --install promstack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace --set nameOverride=promstack --version 40.5.0 --values prometheus-stack-values.yaml
```

```
kubectl --namespace monitoring get pods -l "release=promstack"
k get pods -n monitoring
```

```
cat prometheus-stack-values.yaml
nameOverride: ""

alertmanager:
  enabled: true

  config:
    global:
      resolve_timeout: 5m
    inhibit_rules:
      - source_matchers:
          - 'severity = critical'
        target_matchers:
          - 'severity =~ warning|info'
        equal:
          - 'namespace'
          - 'alertname'
      - source_matchers:
          - 'severity = warning'
        target_matchers:
          - 'severity = info'
        equal:
          - 'namespace'
          - 'alertname'
      - source_matchers:
          - 'alertname = InfoInhibitor'
        target_matchers:
          - 'severity = info'
        equal:
          - 'namespace'
    route:
      group_by: ['namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'null'
      routes:
      - receiver: 'null'
        matchers:
          - alertname =~ "InfoInhibitor|Watchdog"
    receivers:
    - name: 'null'
    templates:
    - '/etc/alertmanager/config/*.tmpl'

  ingress:
    enabled: false
    image:
      repository: quay.io/prometheus/alertmanager
      tag: v0.24.0
    retention: 120h

grafana:
  enabled: true
  adminPassword: prom-operator
  additionalDataSources: 
    - name: Loki
      type: loki
      url: http://loki-query-frontend.logging.svc.cluster.local:3100

```