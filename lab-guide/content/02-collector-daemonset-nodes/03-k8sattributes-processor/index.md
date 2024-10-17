## k8sattributes Processor

### Add `k8sattributes` processor
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

### Validate running pod(s)
Command:
```sh
kubectl get pods -n dynatrace
```
Sample output:
| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| dynatrace-metrics-node-collector-drk1p   | 1/1   | Running | 0        | 1m  |