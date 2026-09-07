# Grafana + Loki Stack on Minikube

Sure. Below is a **from-scratch, copy-paste-friendly setup** for Grafana + Loki on a single-node Minikube cluster.

I recommend using the **Grafana Helm charts** and running Loki in a simple single-binary mode. For a Minikube/lab environment, this is much simpler than building a production Loki cluster.

### Architecture

```text
Applications → Promtail → Loki → Grafana
```

- **Loki** = stores logs
- **Promtail** = collects Kubernetes/container logs
- **Grafana** = UI to search and visualize logs

> **Important:** Grafana's newer observability stack uses **Grafana Alloy** rather than Promtail. Promtail is still useful for understanding the traditional Loki setup, but for a **new deployment in 2026**, I recommend using **Grafana Alloy**. If you specifically want Promtail because you already have it elsewhere, I can give you that version too.

---

# 1. Check your Minikube cluster

First verify that Minikube is running:

```bash
minikube status
```

You should see something similar to:

```text
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

Check Kubernetes:

```bash
kubectl get nodes
```

Expected:

```text
NAME       STATUS   ROLES           AGE
minikube   Ready    control-plane   ...
```

Check resources:

```bash
kubectl top node
```

If `kubectl top` doesn't work, that's okay for now.

---

# 2. Start Minikube with enough resources

If you're creating the cluster from scratch, I'd recommend:

```bash
minikube start \
  --driver=docker \
  --cpus=8 \
  --memory=16g \
  --disk-size=100g
```

For your environment, you previously had a much larger Minikube VM, so you can allocate more if required.

Verify:

```bash
minikube status
kubectl get nodes
```

---

# 3. Enable Minikube addons

Enable Metrics Server:

```bash
minikube addons enable metrics-server
```

Check:

```bash
kubectl get pods -n kube-system
```

You should eventually see:

```text
metrics-server-xxxxx   1/1   Running
```

Test:

```bash
kubectl top nodes
```

---

# 4. Install Helm

Check whether Helm is already installed:

```bash
helm version
```

If not:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

---

# 5. Add Grafana Helm repository

Add the official Grafana Helm repository:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Update repositories:

```bash
helm repo update
```

Check:

```bash
helm repo list
```

You should see:

```text
grafana   https://grafana.github.io/helm-charts
```

---

# 6. Create monitoring namespace

Create a dedicated namespace:

```bash
kubectl create namespace monitoring
```

Verify:

```bash
kubectl get namespace monitoring
```

---

# 7. Install Loki

For a Minikube setup, we'll use Loki in single-binary mode.

Create a values file:

```bash
nano loki-values.yaml
```

Put this inside:

```yaml
deploymentMode: SingleBinary

loki:
  auth_enabled: false

  commonConfig:
    replication_factor: 1

  storage:
    type: filesystem

singleBinary:
  replicas: 1

  persistence:
    enabled: true
    size: 20Gi

gateway:
  enabled: false

read:
  replicas: 0

write:
  replicas: 0

backend:
  replicas: 0

chunksCache:
  enabled: false

resultsCache:
  enabled: false

monitoring:
  selfMonitoring:
    enabled: false
    grafanaAgent:
      installOperator: false

test:
  enabled: false
```

Save:

```text
CTRL + O
ENTER
CTRL + X
```

---

# 8. Install Loki

Run:

```bash
helm upgrade --install loki grafana/loki \
  --namespace monitoring \
  -f loki-values.yaml
```

Check:

```bash
helm list -n monitoring
```

Then:

```bash
kubectl get pods -n monitoring
```

You should eventually get something similar to:

```text
loki-0    1/1    Running
```

---

# 9. Check Loki service

Run:

```bash
kubectl get svc -n monitoring
```

You should see a Loki service.

For example:

```text
NAME        TYPE        CLUSTER-IP      PORT(S)
loki        ClusterIP   10.x.x.x        3100/TCP
```

The important port is:

```text
3100
```

Loki's HTTP API listens on port 3100.

---

# 10. Test Loki directly

Run:

```bash
kubectl port-forward -n monitoring svc/loki 3100:3100
```

Keep this terminal open.

Open another terminal:

```bash
curl http://localhost:3100/ready
```

Expected:

```text
ready
```

You can also check:

```bash
curl http://localhost:3100/metrics
```

If you get Prometheus-style metrics, Loki is working.

Stop the port-forward later with:

```text
CTRL+C
```

---

# 11. Install Grafana Alloy

Now we need something to collect Kubernetes logs and send them to Loki.

Add the Grafana repository if you haven't already:

```bash
helm repo update
```

Install Alloy:

```bash
helm upgrade --install alloy grafana/alloy \
  --namespace monitoring
```

Check:

```bash
kubectl get pods -n monitoring
```

You should see an Alloy pod.

However, **the default Alloy installation doesn't automatically configure Kubernetes log collection**. We need to configure it.

---

# 12. Create Alloy configuration

Create:

```bash
nano alloy-values.yaml
```

Use:

```yaml
alloy:
  configMap:
    create: true
    content: |
      discovery.kubernetes "pods" {
        role = "pod"
      }

      discovery.relabel "pods" {
        targets = discovery.kubernetes.pods.targets

        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          target_label  = "namespace"
        }

        rule {
          source_labels = ["__meta_kubernetes_pod_name"]
          target_label  = "pod"
        }

        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          target_label  = "container"
        }

        rule {
          source_labels = ["__meta_kubernetes_pod_node_name"]
          target_label  = "node"
        }
      }

      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pods.output
        forward_to = [loki.write.default.receiver]
      }

      loki.write "default" {
        endpoint {
          url = "http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push"
        }
      }
```

Save the file.

---

# 13. Upgrade Alloy with the configuration

Run:

```bash
helm upgrade alloy grafana/alloy \
  --namespace monitoring \
  -f alloy-values.yaml
```

Check:

```bash
kubectl get pods -n monitoring
```

You should see something similar to:

```text
alloy-xxxxx   1/1   Running
loki-0        1/1   Running
```

---

# 14. Check Alloy logs

This is an important troubleshooting step.

Run:

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy
```

Look for errors.

You should **not** see errors such as:

```text
connection refused
```

or:

```text
failed to send logs
```

---

# 15. Create a test application

Let's create a simple application that continuously generates logs.

Create:

```bash
nano test-logger.yaml
```

Add:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-logger
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-logger
  template:
    metadata:
      labels:
        app: test-logger
    spec:
      containers:
        - name: logger
          image: busybox
          command:
            - /bin/sh
            - -c
            - |
              while true; do
                echo "$(date) INFO Test application log from Kubernetes"
                sleep 5
              done
```

Apply:

```bash
kubectl apply -f test-logger.yaml
```

Check:

```bash
kubectl get pods
```

You should see:

```text
test-logger-xxxxx   1/1   Running
```

---

# 16. Confirm Kubernetes logs

Run:

```bash
kubectl logs deployment/test-logger
```

You should see:

```text
Mon Sep  7 06:00:00 UTC 2026 INFO Test application log from Kubernetes
Mon Sep  7 06:00:05 UTC 2026 INFO Test application log from Kubernetes
Mon Sep  7 06:00:10 UTC 2026 INFO Test application log from Kubernetes
```

So our application is generating logs.

---

# 17. Check whether Alloy is collecting them

Run:

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy
```

If Alloy is configured correctly, it should be discovering Kubernetes pods and sending their logs to Loki.

---

# 18. Install Grafana

Now install Grafana:

```bash
helm upgrade --install grafana grafana/grafana \
  --namespace monitoring
```

Check:

```bash
kubectl get pods -n monitoring
```

Expected:

```text
grafana-xxxxx   1/1   Running
```

---

# 19. Get Grafana password

Grafana's Helm chart creates a Kubernetes secret.

Run:

```bash
kubectl get secret grafana \
  -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

You will get something like:

```text
xxxxxxxxxxxxxxxx
```

Username:

```text
admin
```

Password:

```text
<password returned above>
```

---

# 20. Access Grafana

Use:

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
```

Then open:

```text
http://localhost:3000
```

Login:

```text
Username: admin
Password: <password>
```

---

# 21. Add Loki as Grafana datasource

Inside Grafana:

```text
Connections
   ↓
Data sources
   ↓
Add data source
   ↓
Loki
```

For the URL use:

```text
http://loki.monitoring.svc.cluster.local:3100
```

Then click:

```text
Save & test
```

You should get:

```text
Data source connected and labels found.
```

---

# 22. Test logs in Grafana

Go to:

```text
Explore
```

Select:

```text
Loki
```

Try this LogQL query:

```logql
{namespace="default"}
```

You should see your `test-logger` logs.

You can narrow it down:

```logql
{namespace="default", pod=~"test-logger.*"}
```

Or:

```logql
{app="test-logger"}
```

---

# 23. Search for ERROR logs

Once your applications are sending logs, you can search:

```logql
{namespace="default"} |= "ERROR"
```

For warnings:

```logql
{namespace="default"} |= "WARN"
```

For a specific application:

```logql
{namespace="augbackend"} |= "ERROR"
```

---

# 24. Check the complete stack

Run:

```bash
kubectl get pods -n monitoring
```

You should have approximately:

```text
NAME                         READY   STATUS
alloy-xxxxx                  1/1     Running
grafana-xxxxx                1/1     Running
loki-0                       1/1     Running
```

Check services:

```bash
kubectl get svc -n monitoring
```

And:

```bash
kubectl get pvc -n monitoring
```

You should have a PVC for Loki:

```text
NAME             STATUS   VOLUME   CAPACITY
storage-loki-0   Bound             20Gi
```

---

# 25. Important: Make Loki data persistent

This is particularly important for your Minikube environment.

Without persistent storage:

```text
Loki
  ↓
Pod deleted
  ↓
Logs can be lost
```

We enabled:

```yaml
persistence:
  enabled: true
  size: 20Gi
```

Check the PVC:

```bash
kubectl get pvc -n monitoring
```

And Minikube storage:

```bash
kubectl get storageclass
```

You should have something similar to:

```text
standard   (default)
```

---

# 26. Useful troubleshooting commands

### Loki

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=loki
```

```bash
kubectl logs -n monitoring loki-0
```

```bash
kubectl describe pod -n monitoring loki-0
```

---

### Alloy

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy
```

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy
```

---

### Grafana

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
```

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana
```

---

### Check everything

```bash
kubectl get all -n monitoring
```

---

# 27. Final architecture

Once everything is working, your Minikube cluster will look like:

```text
                    ┌─────────────────────┐
                    │      Grafana        │
                    │       :3000         │
                    └──────────┬──────────┘
                               │
                               │ LogQL
                               ▼
                    ┌─────────────────────┐
                    │        Loki         │
                    │       :3100         │
                    │                     │
                    │ Persistent Volume   │
                    └──────────┬──────────┘
                               ▲
                               │
                         Push logs
                               │
                    ┌──────────┴──────────┐
                    │       Alloy         │
                    │  DaemonSet/Agents   │
                    └──────────┬──────────┘
                               │
                ┌──────────────┼──────────────┐
                ▼              ▼              ▼
             Pod A           Pod B          Pod C
             logs             logs           logs
```

### Commands you'll use most often

```bash
# Check stack
kubectl get pods -n monitoring

# Check services
kubectl get svc -n monitoring

# Check storage
kubectl get pvc -n monitoring

# Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80

# Loki
kubectl port-forward -n monitoring svc/loki 3100:3100

# Get Grafana password
kubectl get secret grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode

# Check logs
kubectl logs -n monitoring loki-0
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy
```

**One correction to my earlier answer:** I would use **Alloy rather than starting a new Promtail deployment** for a fresh 2026 setup. If your goal is to reproduce the **Grafana + Loki + Promtail** stack you previously had on Minikube, I can also give you a clean **Promtail-based setup with exact YAML files**, including RBAC, DaemonSet, persistence, and troubleshooting for the common `Promtail → Loki` communication issue.
