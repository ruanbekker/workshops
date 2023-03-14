
# Logging

In this workshop we will install grafana loki and promtail.

- Loki: Log Aggregator
- Promtail: Log agent

## Loki

Add the helm repositories:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Search for the versions:

```bash
helm search repo grafana/loki-distributed --versions
```

You will get a list of chart versions that you can choose from:

```bash
NAME                    	CHART VERSION	APP VERSION	DESCRIPTION
grafana/loki-distributed	0.58.0       	2.6.1      	Helm chart for Grafana Loki in microservices mode
grafana/loki-distributed	0.57.0       	2.6.1      	Helm chart for Grafana Loki in microservices mode
```

Dump the default chart values to a file:

```
helm show values grafana/loki-distributed --version 0.58.0 > loki-values.yaml
```

The `loki-values.yaml`:

```yaml
global:
  image:
    registry: null

nameOverride: loki


loki:

  # -- Check https://grafana.com/docs/loki/latest/configuration/#schema_config for more info on how to configure schemas
  schemaConfig:
    configs:
    - from: 2020-09-07
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: loki_index_
        period: 24h

  # -- Check https://grafana.com/docs/loki/latest/configuration/#storage_config for more info on how to configure storages
  storageConfig:
    boltdb_shipper:
      shared_store: filesystem
      active_index_directory: /var/loki/index
      cache_location: /var/loki/cache
      cache_ttl: 168h
    filesystem:
      directory: /var/loki/chunks
# -- Uncomment to configure each storage individually
#   s3: {}
#   boltdb: {}
```

Install loki:

```bash
helm upgrade --install loki grafana/loki-distributed --namespace logging --create-namespace --set nameOverride=loki --version 0.58.0 --values loki-values.yaml
```

View pods:

```
k get pods -n logging
NAME                                   READY   STATUS    RESTARTS   AGE
loki-distributor-cbf4c85fd-xfrkk       1/1     Running   0          2m13s
loki-gateway-6f6ddc78b4-bg9rf          1/1     Running   0          2m13s
loki-ingester-0                        1/1     Running   0          2m13s
loki-querier-0                         1/1     Running   0          2m13s
loki-query-frontend-694bdf9bcb-vzcqk   1/1     Running   0          2m13s
```

View services:

```
k get svc -n logging
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
loki-distributor         ClusterIP   10.96.95.173    <none>        3100/TCP,9095/TCP            17m
loki-gateway             ClusterIP   10.96.123.117   <none>        80/TCP                       17m
loki-ingester            ClusterIP   10.96.196.252   <none>        3100/TCP,9095/TCP            17m
loki-ingester-headless   ClusterIP   None            <none>        3100/TCP,9095/TCP            17m
loki-memberlist          ClusterIP   None            <none>        7946/TCP                     17m
loki-querier             ClusterIP   10.96.145.22    <none>        3100/TCP,9095/TCP            17m
loki-querier-headless    ClusterIP   None            <none>        3100/TCP,9095/TCP            17m
loki-query-frontend      ClusterIP   None            <none>        3100/TCP,9095/TCP,9096/TCP   17m
```

```
loki-distributor.logging.svc.cluster.local
```

## Promtail

Add the helm repositories:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Search for the versions:

```bash
helm search repo grafana/promtail --versions
```

You will get a list of chart versions that you can choose from:

```bash
NAME            	CHART VERSION	APP VERSION	DESCRIPTION
grafana/promtail	6.4.0        	2.6.1      	Promtail is an agent which ships the contents
grafana/promtail	6.3.1        	2.6.1      	Promtail is an agent which ships the contents
```

Dump the default chart values to a file:

```bash
helm show values grafana/promtail --version 6.4.0 > promtail-values.yaml
```

The `promtail-values.yaml`:

```yaml
daemonset:
  enabled: true

config:
  logLevel: info
  serverPort: 3101
  clients:
    - url: http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push
```

Deploy promtail:

```
helm upgrade --install promtail grafana/promtail -f promtail-values.yaml -n logging --create-namespace --set nameOverride=promtail
```

## Minio

```yaml
existingSecret: minio-credentials

defaultBucket:
  enabled: true
  name: loki

DeploymentUpdate:
  type: Recreate

persistence:
  enabled: false
#  size: 10Gi
```

```
helm upgrade minio minio/minio --install --wait --namespace logging --set nameOverride=minio -f minio-values.yaml
```

Loki Values (update ingester, querier, compactor, tablemanager, ruler)

```yaml
global:


loki:

  # -- Check https://grafana.com/docs/loki/latest/configuration/#schema_config for more info on how to configure schemas
  schemaConfig:
    configs:
    - from: 2020-09-07
      store: boltdb-shipper
    #   object_store: filesystem
      object_store: s3
      schema: v11
      index:
        prefix: loki_index_
        period: 24h

  # -- Check https://grafana.com/docs/loki/latest/configuration/#storage_config for more info on how to configure storages
  storageConfig:
    boltdb_shipper:
    #   shared_store: filesystem
      shared_store: s3
      active_index_directory: /var/loki/index
      cache_location: /var/loki/cache
      cache_ttl: 168h
    aws:
    #   s3: http://minio.logging.svc.cluster.local.:9000/loki
    #   s3forcepathstyle: true
      endpoint: minio.logging.svc.cluster.local:9000
      region: us-east-1
      s3forcepathstyle: true
      insecure: true
      bucketnames: loki
      access_key_id: fddb60a94042b193ded75d9cda5d338f0cf901e3
      secret_access_key: 35a090b47bafb4eac7b1d9d0228f440decefdd8253e707298c17fc346f565008c846769f4e444a41
      s3forcepathstyle: true
# Configuration for the ingester
ingester:
  kind: StatefulSet
  replicas: 1
  autoscaling:
    enabled: false
  extraEnvFrom:
    - secretRef:
        name: minio-credentials

# Configuration for the distributor
distributor:
  replicas: 1
  autoscaling:
    enabled: false

# Configuration for the querier
querier:
  replicas: 1
  extraEnvFrom:
    - secretRef:
        name: minio-credentials

# Configuration for the query-frontend
queryFrontend:
  # -- Number of replicas for the query-frontend
  replicas: 1


# Configuration for the gateway
gateway:
  # -- Specifies whether the gateway should be enabled
  enabled: true
  # -- Number of replicas for the gateway
  replicas: 1
  # -- Enable logging of 2xx and 3xx HTTP requests

# Configuration for the compactor
compactor:
  # -- Specifies whether compactor should be enabled
  enabled: false
# Configuration for the ruler
ruler:
  # -- Specifies whether the ruler should be enabled
  enabled: false

# Configuration for the index-gateway
indexGateway:
  # -- Specifies whether the index-gateway should be enabled
  enabled: false
```

```
helm upgrade --install loki grafana/loki-distributed --namespace logging --create-namespace --set nameOverride=loki --version 0.58.0 --values loki-values.yaml
```

```
helm repo add minio https://helm.min.io
kubectl create secret generic --namespace=logging minio-credentials --from-literal=accesskey="$(openssl rand -hex 20)" --from-literal=secretkey="$(openssl rand -hex 40)"
helm upgrade minio minio/minio --install --wait --namespace logging --set nameOverride=minio -f minio-values.yaml
echo $(kubectl get secret minio-credentials -n logging -o jsonpath="{.data.accesskey}" | base64 --decode)
echo $(kubectl get secret minio-credentials -o jsonpath="{.data.secretkey}" | base64 --decode)
```



- https://github.com/unguiculus/loki-minio-demo
- https://www.sobyte.net/post/2022-06/grafana-loki-rw/

## Grafana

To configure grafana: `http://loki-query-frontend.logging.svc.cluster.local:3100`