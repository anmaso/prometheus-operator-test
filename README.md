# prometheus-operator-test

This is a test to configure prometheus to scrap metrics from velero
It uses a k8s kind cluster for tests

## Requirements:

- docker or podman installed and working
- kubectl
- git
- kind
- velero CLI

## Setup

Check if everything is installing before attempting to install again

1. **Provision the cluster**

   ```bash
   kind create cluster
   ```

2. **Install Velero CLI**

   _Using Homebrew (macOS):_

   ```bash
   brew install velero
   ```

3. **Clone the Velero repository**

   ```bash
   git clone https://github.com/vmware-tanzu/velero
   ```

   deploy a test workload

   ```

   kubectl apply -f examples/nginx-app/base.yaml

   ```

4. **Create Velero credentials file**

   Create a file named `credentials-velero` with the following content:

   ```bash
   cat << EOF > credentials-velero
   [default]
   aws_access_key_id = minio
   aws_secret_access_key = minio123
   EOF
   ```

5. **Deploy Minio**

   ```bash
   kubectl apply -f velero/examples/minio/00-minio-deployment.yaml
   ```

   We need to create a velero bucket in minio, the setup pod fails for some reason and we have to do it manually
   The minio pod contains a webUI that can be port-fowwarded and the bucket created manually

6. **Install Velero**

   ```bash
   ./velero/velero install \
       --provider aws \
       --plugins velero/velero-plugin-for-aws:v1.2.1 \
       --bucket velero \
       --secret-file ./credentials-velero \
       --use-volume-snapshots=false \
       --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000
   ```

7. **Install Prometheus Operator**

   We will use the `release-0.10` branch of `kube-prometheus` for compatibility with the default `kind` Kubernetes version.

   ```bash
   git clone --branch release-0.10 https://github.com/prometheus-operator/kube-prometheus.git
   kubectl create -f kube-prometheus/manifests/setup
   until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
   kubectl create -f kube-prometheus/manifests/
   ```

8. **Configure Prometheus to scrape Velero**

   a. **Create a Role and RoleBinding for Prometheus**

   Create a file named `role.yaml`:

   ```yaml
   cat << EOF > role.yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     namespace: velero
     name: velero-reader
   rules:
    - apiGroups: [""]
      resources: ["pods", "services']
      verbs: ["get", "list", "watch"]
   EOF
   ```

   Create a file named `rolebinding.yaml`:

   ```yaml
   cat << EOF > rolebinding.yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: velero-binding
     namespace: velero
   subjects:
     - kind: ServiceAccount
       name: prometheus-k8s
       namespace: monitoring
   roleRef:
     kind: Role
     name: velero-reader
     apiGroup: rbac.authorization.k8s.io
   EOF
   ```

   Apply the files:

   ```bash
   kubectl apply -f role.yaml
   kubectl apply -f rolebinding.yaml
   ```

   b. **Create a Service for Velero**

   ```bash
   kubectl create service clusterip velero --tcp=8085:8085 -n velero
   kubectl patch service velero -n velero -p '{"spec":{"selector":{"component": "velero"}}}'
   kubectl label service velero -n velero app=velero --overwrite
   ```

   c. **Create a ServiceMonitor**

   Create a file named `servicemonitor.yaml`:

   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata:
     name: velero-sm
     namespace: monitoring
   spec:
     selector:
       matchLabels:
         app: velero
     namespaceSelector:
       matchNames:
         - velero
     endpoints:
       - port: metrics
         path: /metrics
   ```

   Apply the file:

   ```bash
   kubectl apply -f servicemonitor.yaml
   ```

   d. **Configure Prometheus to select the ServiceMonitor**

   ```bash
   kubectl patch prometheus k8s -n monitoring --type='json' -p='[{"op": "add", "path": "/spec/serviceMonitorSelector", "value": {}}]'
   ```

9. **Create a test backup and a scheduled one**

Adjust the time to the next minute, not to wait too much for it to happen

```bash
velero backup create nginx-backup --selector app=nginx
velero schedule create nginx-daily --schedule="1 * * * *" --selector app=nginx

```

10. **Query Prometheus for Velero metrics**

    ```bash
    kubectl port-forward -n monitoring service/prometheus-k8s 9090:9090 &
    curl -s http://localhost:9090/api/v1/query?query=velero_backup_success_total | jq
    ```

11. **Test deleting the nginx namespace completely, we will recover it with restore**
    ```bash
    kubectl delete namespace nginx
    ```
12. **Recover the nginx namespace**

    ```bash
    velero restore create --from-backup nginx-backup
    velero restore get

    ```

## Troubleshooting

If you are unable to scrape the Velero metrics, you can try the following:

- **Check the Prometheus targets:**
  ```bash
  curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.scrapePool == "service-monitor/monitoring/velero-sm/0")'
  ```
- **Check the Prometheus Operator logs:**
  ```bash
  kubectl logs -n monitoring deploy/prometheus-operator
  ```
- **Restart the Prometheus pods:**
  ```bash
  kubectl delete pod -n monitoring -l app.kubernetes.io/name=prometheus
  ```
- **Ensure the `serviceMonitorSelector` in the `Prometheus` custom resource is set to `{}`.**
  ```bash
  kubectl get prometheus k8s -n monitoring -o jsonpath='{.spec.serviceMonitorSelector}'
  ```
- **Ensure the `ServiceMonitor`'s selector matches the labels on the `velero` service.**
- **Ensure the `velero` service's selector matches the labels on the `velero` pod.**
