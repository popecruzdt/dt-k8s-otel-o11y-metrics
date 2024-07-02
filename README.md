id: dt-k8s-otel-o11y-metrics
summary: dynatrace metric ingest for kubernetes using opentelemetry collector
author: Tony Pope-Cruz

# Kubernetes Log Ingest with OpenTelemetry & Dynatrace
<!-- ------------------------ -->
## Overview 
Total Duration: 20

### What You’ll Learn Today
In this lab we'll utilize the OpenTelemetry Collector deployed as a DaemonSet (Node Agent) to collect Node (kubelet) metrics from a Kubernetes cluster and ship them to Dynatrace.  Additionally, we'll utilize a second OpenTelemetry Collector deployed as a Deployment (Gateway) to collect Cluster (Kubernetes API) metrics from the Kubernetes cluster and ship them to Dynatrace.

Lab tasks:
1. Create a Kubernetes cluster on Google GKE
2. Deploy OpenTelemetry's demo application, astronomy-shop
3. Deploy OpenTelemetry Collector as a DaemonSet
4. Configure OpenTelemetry Collector service pipeline for metric enrichment
5. Deploy OpenTelemetry Collector as a Deployment
6. Configure OpenTelemetry Collector service pipeline for metric enrichment
5. Query and visualize metrics in Dynatrace using DQL

![astronomy-shop logs](img/astronomy-shop_logs.png)

<!-- -------------------------->
## Technical Specification 
Duration: 2

#### Technologies Used
- [Dynatrace](https://www.dynatrace.com/trial)
- [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine)
  - tested on GKE v1.29.4-gke.1043002
- [OpenTelemetry Demo astronomy-shop](https://opentelemetry.io/docs/demo/)
  - tested on release 1.10.0
- [Istio](https://istio.io/latest/docs/)
  - tested on v1.22.1
- [OpenTelemetry Collector - Contrib Distro](https://github.com/open-telemetry/opentelemetry-collector-contrib/releases)
  - tested on v0.103.0

#### Reference Architecture
TODO

#### Prerequisites
- Google Cloud Account
- Google Cloud Project
- Google Cloud Access to Create and Manage GKE Clusters
- Google CloudShell Access

<!-- -------------------------->
## Setup
Duration: 18

### Prerequisites

#### Deploy GKE cluster & demo assets
https://github.com/popecruzdt/dt-k8s-otel-o11y-cluster

#### Clone the repo to your home directory
Command:
```sh
git clone https://github.com/popecruzdt/dt-k8s-otel-o11y-metrics.git
```
Sample output:
> Cloning into 'dt-k8s-otel-o11y-metrics'...\
> ...\
> Receiving objects: 100% (12/12), 10.61 KiB | 1.77 MiB/s, done.

#### Move into repo base directory
Command:
```sh
cd dt-k8s-otel-o11y-metrics
```

#### Generate Dynatrace Access Token
Generate a new API access token with the following scopes:
```
Ingest events
Ingest logs
Ingest metrics
Ingest OpenTelemetry traces
```
[See Related Dynatrace API Token Creation Documentation](https://docs.dynatrace.com/docs/dynatrace-api/basics/dynatrace-api-authentication#create-token)
![dt access token](img/dt_access_token.png)

#### (optional) Import Notebook into Dynatrace
[notebook](/k8s-otel-o11y-metrics_dt_notebook.json)

#### Define workshop user variables
In your GCP CloudShell Terminal:
```
DT_ENDPOINT=https://{your-environment-id}.live.dynatrace.com/api/v2/otlp
DT_API_TOKEN={your-api-token}
NAME={initials}-k8s-otel-o11y
```
### OpenTelemetry Collector - Contrib Distro
https://github.com/open-telemetry/opentelemetry-collector-contrib

#### Create `dynatrace` namespace
Command:
```sh
kubectl create namespace dynatrace
```
Sample output:
> namespace/dynatrace created

#### Create `dynatrace-otelcol-dt-api-credentials` secret
Command:
```sh
kubectl create secret generic dynatrace-otelcol-dt-api-credentials --from-literal=DT_ENDPOINT=$DT_ENDPOINT --from-literal=DT_API_TOKEN=$DT_API_TOKEN -n dynatrace
```
Sample output:
> secret/dynatrace-otelcol-dt-api-credentials created

#### Deploy `cert-manager`, pre-requisite for `opentelemetry-operator`
https://cert-manager.io/docs/installation/

Command:
```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```
Sample output:
> namespace/cert-manager created\
> customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created\
> customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created\
> ...\
> validatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created

#### Deploy `opentelemetry-operator`
Command:
```sh
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```
Sample output:
> namespace/opentelemetry-operator-system created\
> customresourcedefinition.apiextensions.k8s.io/instrumentations.opentelemetry.io created\
> customresourcedefinition.apiextensions.k8s.io/opampbridges.opentelemetry.io created\
> ...\
> validatingwebhookconfiguration.admissionregistration.k8s.io/opentelemetry-operator-validating-webhook-configuration configured

#### Deploy OpenTelemetry Collector - Contrib Distro - Daemonset (Node Agent)
https://github.com/open-telemetry/opentelemetry-operator
```yaml
---
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: dynatrace-metrics-node
  namespace: dynatrace
spec:
  envFrom:
  - secretRef:
      name: dynatrace-otelcol-dt-api-credentials
  env:
    - name: K8S_NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
  mode: "daemonset"
  image: "otel/opentelemetry-collector-contrib:0.103.0"
```
Command:
```sh
kubectl apply -f opentelemetry/collector/metrics/otel-collector-metrics-node-crd-01.yaml
```
Sample output:
> opentelemetrycollector.opentelemetry.io/dynatrace-metrics-node created

##### Validate running pod(s)
Command:
```sh
kubectl get pods -n dynatrace
```
Sample output:
| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| dynatrace-logs-collector-8q8tz   | 1/1   | Running | 0        | 1m  |

##### Create `clusterrole` with read access to Kubernetes objects
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-k8s-clusterrole-metrics
rules:
  - apiGroups: ['']
    resources: ['events', 'namespaces', 'namespaces/status', 'nodes', 'nodes/spec', 'nodes/stats', 'nodes/proxy', 'pods', 'pods/status', 'replicationcontrollers', 'replicationcontrollers/status', 'resourcequotas', 'services']
    verbs: ['get', 'list', 'watch']
  - apiGroups: ['apps']
    resources: ['daemonsets', 'deployments', 'replicasets', 'statefulsets']
    verbs: ['get', 'list', 'watch']
  - apiGroups: ['extensions']
    resources: ['daemonsets', 'deployments', 'replicasets']
    verbs: ['get', 'list', 'watch']
  - apiGroups: ['batch']
    resources: ['jobs', 'cronjobs']
    verbs: ['get', 'list', 'watch']
  - apiGroups: ['autoscaling']
    resources: ['horizontalpodautoscalers']
    verbs: ['get', 'list', 'watch']
```
Command:
```sh
kubectl apply -f opentelemetry/rbac/otel-collector-k8s-clusterrole-metrics.yaml
```
Sample output:
> clusterrole.rbac.authorization.k8s.io/otel-collector-k8s-clusterrole-metrics created

##### Create `clusterrolebinding` for OpenTelemetry Collector service account
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-k8s-clusterrole-metrics-crb
subjects:
- kind: ServiceAccount
  name: dynatrace-metrics-node-collector
  namespace: dynatrace
roleRef:
  kind: ClusterRole
  name: otel-collector-k8s-clusterrole-metrics
  apiGroup: rbac.authorization.k8s.io
```
Command:
```sh
kubectl apply -f opentelemetry/rbac/otel-collector-k8s-clusterrole-metrics-crb.yaml
```
Sample output:
> clusterrolebinding.rbac.authorization.k8s.io/otel-collector-k8s-clusterrole-metrics-crb created

##### `kubeletstats` receiver
https://opentelemetry.io/docs/kubernetes/collector/components/#kubeletstats-receiver
```yaml
config: |
    receivers:
      kubeletstats:
        collection_interval: 30s
        auth_type: 'serviceAccount'
        endpoint: '${env:K8S_NODE_NAME}:10250'
        insecure_skip_verify: true
        metric_groups:
          - node
          - pod
          - container
```
Default Metrics:\
https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/kubeletstatsreceiver/documentation.md

##### Query Node metrics in Dynatrace
DQL:
```sql
timeseries node_cpu = avg(k8s.node.cpu.utilization), by: {k8s.cluster.name, k8s.node.name}
```
Result:\
![dql_kubeletstats_node_cpu](img/dql_kubeletstats_node_cpu.png)



##### Add `k8sattributes` processor
https://opentelemetry.io/docs/kubernetes/collector/components/#kubernetes-attributes-processor
```yaml
k8sattributes:
    auth_type: "serviceAccount"
    passthrough: false
        filter:
        node_from_env_var: KUBE_NODE_NAME
    extract:
        metadata:
            - k8s.namespace.name
            - k8s.deployment.name
            - k8s.daemonset.name
            - k8s.job.name
            - k8s.cronjob.name
            - k8s.replicaset.name
            - k8s.statefulset.name
            - k8s.pod.name
            - k8s.pod.uid
            - k8s.node.name
            - k8s.container.name
            - container.id
            - container.image.name
            - container.image.tag
        labels:
        - tag_name: app.label.component
            key: app.kubernetes.io/component
            from: pod
    pod_association:
        - sources:
            - from: resource_attribute
              name: k8s.pod.uid
        - sources:
            - from: resource_attribute
              name: k8s.pod.name
        - sources:
            - from: resource_attribute
              name: k8s.pod.ip
        - sources:
            - from: connection
```
Command:
```sh
kubectl apply -f opentelemetry/collector/metrics/otel-collector-metrics-node-crd-02.yaml
```
Sample output:
> opentelemetrycollector.opentelemetry.io/dynatrace-metrics-node configured

##### Validate running pod(s)
Command:
```sh
kubectl get pods -n dynatrace
```
Sample output:
| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| dynatrace-logs-collector-dns4x   | 1/1   | Running | 0        | 1m  |

##### Query logs in Dynatrace
DQL:
```sql
fetch logs
| filter k8s.namespace.name == "astronomy-shop" and isNotNull(k8s.deployment.name)
| sort timestamp desc
| limit 100
| fields timestamp, loglevel, status, k8s.namespace.name, k8s.deployment.name, k8s.pod.name, k8s.container.name, app.label.component, content
```
Result:\
![dql_k8sattributes_processor](img/dql_k8sattributes_processor.png)

#### Deploy OpenTelemetry Collector - Contrib Distro - Deployment (Gateway)
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

##### Validate running pod(s)
Command:
```sh
kubectl get pods -n dynatrace
```
Sample output:
| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| dynatrace-logs-collector-8q8tz   | 1/1   | Running | 0        | 1m  |

##### `k8s_cluster` receiver
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
Default Metrics:\
https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/k8sclusterreceiver/documentation.md

##### Query Deployment metrics in Dynatrace
DQL:
```sql
timeseries node_cpu = min(k8s.deployment.available), by: {k8s.cluster.name, k8s.deployment.name}, filter: {k8s.namespace.name == "astronomy-shop"}
```
Result:\
![dql_k8scluster_pod_avail](img/dql_k8scluster_pod_avail.png)

### Export OpenTelemetry data from `astronomy-shop` to OpenTelemetry Collector - Contrib Distro

#### Customize astronomy-shop helm values
```yaml
default:
  # List of environment variables applied to all components
  env:
    - name: OTEL_SERVICE_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: "metadata.labels['app.kubernetes.io/component']"
    - name: OTEL_COLLECTOR_NAME
      value: '{{ include "otel-demo.name" . }}-otelcol'
    - name: OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      value: cumulative
    - name: OTEL_RESOURCE_ATTRIBUTES
      value: 'service.name=$(OTEL_SERVICE_NAME),service.namespace=NAME_TO_REPLACE,service.version={{ .Chart.AppVersion }}'
```
> service.namespace=NAME_TO_REPLACE\
> service.namespace=INITIALS-k8s-otel-o11y

Command:
```sh
sed -i "s,NAME_TO_REPLACE,$NAME," astronomy-shop/collector-values.yaml
```

#### Update `astronomy-shop` OpenTelemetry Collector export endpoint via helm
Command:
```sh
helm upgrade astronomy-shop open-telemetry/opentelemetry-demo --values astronomy-shop/collector-values.yaml --namespace astronomy-shop
```
Sample output:
> NAME: astronomy-shop\
> LAST DEPLOYED: Thu Jun 27 20:58:38 2024\
> NAMESPACE: astronomy-shop\
> STATUS: deployed\
> REVISION: 2

#### Query `astronomy-shop` metrics in Dynatrace
DQL:
```sql
timeseries jvm_mem_used = avg(jvm.memory.used), by: {service.name, k8s.cluster.name}, filter: {k8s.namespace.name == "astronomy-shop"}
```
Result:\
![dql_kubeletstats_node_cpu](img/dql_kubeletstats_node_cpu.png)

DQL:
```sql
timeseries avg(kafka.consumer.request_rate), by: {service.name, k8s.cluster.name}, filter: {k8s.namespace.name == "astronomy-shop"}
```
Result:\
![dql_kubeletstats_node_cpu](img/dql_kubeletstats_node_cpu.png)

<!-- ------------------------ -->
## Demo The New Functionality
TODO

<!-- -------------------------->
## Wrap Up

### What You Learned Today 
By completing this lab, you've successfully deployed the OpenTelemetry Collector to collect metrics, enrich metric attributes for better context, and ship those metrics to Dynatrace for analysis.
- One Community Contrib Distro OpenTelemetry Collector was deployed as a DaemonSet, behaving as an Agent running on each Node
  - The `kubeletstats` receiver scrapes metrics from the local kubelet on the Node
  - The `k8sattributes` processor enriches the metrics with Kubernetes attributes that may be missing without it
- A second Community Contrib Distro OpenTelemetry Collector was deployed as a Deployment, behaving as a Gateway
  - The `k8s_cluster` receiver queries the Kubernetes cluster API to retrieve metrics
  - The `k8sattributes` processor enriches the metrics with Kubernetes attributes that may be missing without it
- Metrics produced by the OpenTelemetry SDKs and Agents are exported to the `otlp` receiver
- Dynatrace DQL (via Notebooks) allows you to perform powerful queries and analysis of the metric data

<!-- ------------------------ -->
### Supplemental Material
TODO
