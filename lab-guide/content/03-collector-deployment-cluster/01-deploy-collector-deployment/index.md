## Deploy OpenTelemetry Collector

### Contrib Distro - Deployment (Gateway)
https://github.com/open-telemetry/opentelemetry-operator
```yaml
---
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: dynatrace-metrics-cluster
  namespace: dynatrace
spec:
  envFrom:
  - secretRef:
      name: dynatrace-otelcol-dt-api-credentials
  mode: "deployment"
  image: "otel/opentelemetry-collector-contrib:0.103.0"
```
Command:
```sh
kubectl apply -f opentelemetry/collector/metrics/otel-collector-metrics-cluster-crd-01.yaml
```
Sample output:
> opentelemetrycollector.opentelemetry.io/dynatrace-metrics-cluster created

### Validate running pod(s)
Command:
```sh
kubectl get pods -n dynatrace
```
Sample output:
| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| dynatrace-metrics-cluster-collector-7bd8dc4995-6sgs2   | 1/1   | Running | 0        | 1m  |

### `k8s_cluster` receiver
https://opentelemetry.io/docs/kubernetes/collector/components/#kubernetes-cluster-receiver
```yaml
config: |
    receivers:
      k8s_cluster:
        collection_interval: 60s
        node_conditions_to_report: [ "Ready", "MemoryPressure", "DiskPressure" ]
        allocatable_types_to_report: [ "cpu","memory" ]
        metadata_collection_interval: 5m
```
Default Metrics:

https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/k8sclusterreceiver/documentation.md

### Query Deployment metrics in Dynatrace
DQL:
```sql
timeseries pods_avail = min(k8s.deployment.available), by: {k8s.cluster.name, k8s.deployment.name}, filter: {k8s.namespace.name == "astronomy-shop"}
```
Result:

![dql_k8scluster_pod_avail](../../../assets/images/03-dql_k8scluster_pod_avail.png)