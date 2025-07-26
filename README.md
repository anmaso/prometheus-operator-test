# prometheus-operator-test
#
This is a test to configure prometheus to scrap metrics from velero
It uses a k8s kind cluster for tests

Requirements:
- docker or podman installed and working
- kubectl
- git
- kind
- velero CLI

start by provisioning the cluster

```
kind create cluster
```

verify access to the cluster 

Clone the velero repository

```
git clone https://github.com/vmware-tanzu/velero
```

Create a Velero-specific credentials file (credentials-velero) in your Velero directory:

```
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
```

deploy a sample workload

```
kubectl apply -f examples/minio/00-minio-deployment.yaml
```

install velero


```
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.2.1 \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000
```


To install the prometheus operator 

```
git clone https://github.com/prometheus-operator/kube-prometheus.git

  # Create the namespace and CRDs, and then wait for them to be available before creating the remaining resources
kubectl create -f manifests/setup

# Wait until the "servicemonitors" CRD is created. The message "No resources found" means success in this context.
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done

kubectl create -f manifests/
```


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



