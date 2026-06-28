# Kubernetes Monitoring Stack Project Commands

This file documents the commands used during the kube-prometheus-stack monitoring Project.

> This is not an executable script.
> Run commands manually step by step when needed.

---

## 1. Cluster Pre-Checks

Check Kubernetes version:

```bash
kubectl version
```

Check cluster nodes:

```bash
kubectl get nodes -o wide
```

Check all running pods:

```bash
kubectl get pods -A
```

Check cluster permissions:

```bash
kubectl auth can-i '*' '*' --all-namespaces
```

Expected result:

```text
yes
```

---

## 2. Helm Repository Configuration

Add the Prometheus Community Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Update Helm repositories:

```bash
helm repo update
```

Search for the kube-prometheus-stack chart:

```bash
helm search repo prometheus-community/kube-prometheus-stack
```

List available chart versions:

```bash
helm search repo prometheus-community/kube-prometheus-stack --versions | head
```

---

## 3. Create Monitoring Namespace

Create the monitoring namespace:

```bash
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
```

Verify the namespace:

```bash
kubectl get namespace monitoring
```

---

## 4. Install kube-prometheus-stack

Install the monitoring stack:

```bash
helm upgrade --install monitoring-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --wait \
  --timeout 15m
```

Verify the Helm release:

```bash
helm list -n monitoring
```

Check release status:

```bash
helm status monitoring-stack -n monitoring
```

Check release history:

```bash
helm history monitoring-stack -n monitoring
```

---

## 5. Verify Monitoring Resources

Check pods:

```bash
kubectl get pods -n monitoring -o wide
```

Check deployments:

```bash
kubectl get deployments -n monitoring
```

Check StatefulSets:

```bash
kubectl get statefulsets -n monitoring
```

Check DaemonSets:

```bash
kubectl get daemonsets -n monitoring
```

Check services:

```bash
kubectl get services -n monitoring
```

Check ConfigMaps:

```bash
kubectl get configmaps -n monitoring
```

Check Secrets:

```bash
kubectl get secrets -n monitoring
```

Check ServiceAccounts:

```bash
kubectl get serviceaccounts -n monitoring
```

---

## 6. Check Prometheus Operator Custom Resources

List Prometheus Operator CRDs:

```bash
kubectl get crd | grep monitoring.coreos.com
```

List monitoring API resources:

```bash
kubectl api-resources --api-group=monitoring.coreos.com
```

List Prometheus resources:

```bash
kubectl get prometheus -A
```

List Alertmanager resources:

```bash
kubectl get alertmanager -A
```

List ServiceMonitor resources:

```bash
kubectl get servicemonitor -A
```

List PodMonitor resources:

```bash
kubectl get podmonitor -A
```

List PrometheusRule resources:

```bash
kubectl get prometheusrule -A
```

---

## 7. Access Grafana

Port-forward Grafana:

```bash
kubectl -n monitoring port-forward svc/monitoring-stack-grafana 3000:80
```

Open Grafana locally:

```text
http://localhost:3000
```

Retrieve Grafana admin username:

```bash
kubectl get secret -n monitoring monitoring-stack-grafana \
  -o jsonpath="{.data.admin-user}" | base64 -d; echo
```

Retrieve Grafana admin password:

```bash
kubectl get secret -n monitoring monitoring-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

For EC2 access, use an SSH tunnel from the local machine:

```bash
ssh -i <SSH_KEY_FILE> -L 3000:127.0.0.1:3000 <USER>@<MASTER_PUBLIC_IP>
```

Then open:

```text
http://localhost:3000
```

---

## 8. Access Prometheus

Port-forward Prometheus:

```bash
kubectl -n monitoring port-forward svc/monitoring-stack-kube-prom-prometheus 9090:9090
```

Open Prometheus locally:

```text
http://localhost:9090
```

Open Prometheus Targets page:

```text
http://localhost:9090/targets
```

Open Prometheus Service Discovery page:

```text
http://localhost:9090/service-discovery
```

For EC2 access, use an SSH tunnel:

```bash
ssh -i <SSH_KEY_FILE> -L 9090:127.0.0.1:9090 <USER>@<MASTER_PUBLIC_IP>
```

---

## 9. Useful PromQL Queries

Check all targets:

```promql
up
```

Check healthy targets grouped by job:

```promql
sum by (job) (up)
```

Check Kubernetes node information:

```promql
kube_node_info
```

Check node exporter system information:

```promql
node_uname_info
```

Check Prometheus build information:

```promql
prometheus_build_info
```

Check running pods by namespace:

```promql
sum by (namespace) (kube_pod_status_phase{phase="Running"})
```

Check node CPU usage:

```promql
100 * (1 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))
```

Check node memory usage:

```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```

---

## 10. Deploy Sample Application

Apply the sample app manifest:

```bash
kubectl apply -f manifests/sample-app.yaml
```

Verify the app pods:

```bash
kubectl get pods -n demo-app -o wide
```

Verify the app service:

```bash
kubectl get svc -n demo-app
```

---

## 11. Deploy ServiceMonitor for Sample Application

Apply the ServiceMonitor manifest:

```bash
kubectl apply -f manifests/servicemonitor.yaml
```

Verify the ServiceMonitor:

```bash
kubectl get servicemonitor -n monitoring example-app
```

Check Prometheus Targets page and search for:

```text
example-app
```

---

## 12. Troubleshooting Commands Used During the Lab

Check kube-controller-manager metrics listener:

```bash
sudo ss -lntp | grep 10257
```

Check kube-scheduler metrics listener:

```bash
sudo ss -lntp | grep 10259
```

Check etcd metrics listener:

```bash
sudo ss -lntp | grep 2381
```

Check kube-controller-manager bind address:

```bash
sudo grep -n -- 'bind-address' /etc/kubernetes/manifests/kube-controller-manager.yaml
```

Check kube-scheduler bind address:

```bash
sudo grep -n -- 'bind-address' /etc/kubernetes/manifests/kube-scheduler.yaml
```

Check etcd metrics URL:

```bash
sudo grep -n -- 'listen-metrics-urls' /etc/kubernetes/manifests/etcd.yaml
```

---

## 13. Fix kube-controller-manager Metrics Binding

Problem:

```text
Prometheus showed kube-controller-manager target as DOWN.
```

Reason:

```text
kube-controller-manager was listening only on 127.0.0.1:10257.
```

Fix:

```bash
sudo sed -i 's/--bind-address=127.0.0.1/--bind-address=0.0.0.0/' \
/etc/kubernetes/manifests/kube-controller-manager.yaml
```

Restart kubelet:

```bash
sudo systemctl restart kubelet
```

Verify:

```bash
sudo ss -lntp | grep 10257
```

Expected result:

```text
0.0.0.0:10257
```

---

## 14. Fix kube-scheduler Metrics Binding

Problem:

```text
Prometheus showed kube-scheduler target as DOWN.
```

Reason:

```text
kube-scheduler was listening only on 127.0.0.1:10259.
```

Fix:

```bash
sudo sed -i 's/--bind-address=127.0.0.1/--bind-address=0.0.0.0/' \
/etc/kubernetes/manifests/kube-scheduler.yaml
```

If the static pod does not restart automatically, force recreation:

```bash
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/kube-scheduler.yaml.fixed
sudo mv /tmp/kube-scheduler.yaml.fixed /etc/kubernetes/manifests/kube-scheduler.yaml
```

Verify:

```bash
sudo ss -lntp | grep 10259
```

Expected result:

```text
0.0.0.0:10259
```

---

## 15. Fix etcd Metrics Binding

Problem:

```text
Prometheus showed kube-etcd target as DOWN.
```

Reason:

```text
etcd metrics were listening only on 127.0.0.1:2381.
```

Fix:

```bash
sudo sed -i 's|--listen-metrics-urls=http://127.0.0.1:2381|--listen-metrics-urls=http://0.0.0.0:2381|' \
/etc/kubernetes/manifests/etcd.yaml
```

If the static pod does not restart automatically, force recreation:

```bash
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.fixed
sudo mv /tmp/etcd.yaml.fixed /etc/kubernetes/manifests/etcd.yaml
```

Verify:

```bash
sudo ss -lntp | grep 2381
```

Expected result:

```text
0.0.0.0:2381
```

---

## 16. Security Notes

Do not commit any of the following:

```text
Real EC2 public IPs
Real private node IPs
SSH private keys
Kubernetes kubeconfig files
admin.conf
Tokens
Passwords
Secrets
Cloud credentials
```

Do not expose these metrics ports publicly in the EC2 Security Group:

```text
10257
10259
10249
2381
```

They should only be reachable inside the private cluster network.

---

## 17. Final Verification

Check all monitoring pods:

```bash
kubectl get pods -n monitoring
```

Check Prometheus targets:

```text
http://localhost:9090/targets
```

Expected important targets:

```text
Kubernetes API Server: UP
kubelet: UP
CoreDNS: UP
kube-state-metrics: UP
Node Exporter: UP
Prometheus Operator: UP
Prometheus Server: UP
Grafana: UP
Alertmanager: UP
kube-controller-manager: UP
kube-scheduler: UP
kube-etcd: UP
```

