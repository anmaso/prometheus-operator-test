# prometheus-operator-test
#
#

used this one
git clone https://github.com/prometheus-operator/kube-prometheus.git


NOT this one
git clone https://github.com/prometheus-operator/prometheus-operator.git

some examples in examples/user-guide/getting-started

```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
```

To access apps in another namespace, Prometheus service account will need permission in another namespace to access pods

  - role in target namespace
  - binding in target namespace pointing to the service account in the prometheus monitoring namespace

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: velero
  name: pod-reader
rules:
 - apiGroups: [""]
   resources: ["pods"]
   verbs: ["get", "list", "watch"]
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: velero
subjects:
  - kind: ServiceAccount
    name: prometheus-k8s
    namespace: monitoring
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

example podMonitor to read velero metrics 

```
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: velero-pod
  namespace: monitoring
spec:
  selector:
    matchLabels:
      component: velero
      deploy: velero
  namespaceSelector:
    matchNames:
      - velero
  podMetricsEndpoints:
  - port: metrics

```
