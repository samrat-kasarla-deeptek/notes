To enable logging and monitoring for an Amazon Web Services Amazon EKS Fargate cluster using Amazon CloudWatch, you mainly configure:

1. **Container Insights** (metrics)
2. **Fluent Bit logging** (application/container logs)

For EKS Fargate, you **cannot run DaemonSets**, so AWS provides a built-in Fluent Bit integration.

---

# 1. Prerequisites

Ensure:

* EKS cluster is already running on Fargate
* `kubectl` configured
* AWS CLI configured
* OIDC enabled on cluster

Check cluster:

```bash
aws eks describe-cluster --name <cluster-name> --region <region>
```

---

# 2. Enable IAM OIDC Provider

Check existing provider:

```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

If not enabled:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster <cluster-name> \
  --approve
```

---

# 3. Enable CloudWatch Container Insights (Metrics)

Create IAM policy attachment for CloudWatch.

## Create IAM Role for CloudWatch Agent

Download policy:

```bash
curl -O https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-onboard.sh
```

But for Fargate, simpler approach:

Create service account:

```bash
eksctl create iamserviceaccount \
  --name cloudwatch-agent \
  --namespace amazon-cloudwatch \
  --cluster <cluster-name> \
  --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
  --approve \
  --override-existing-serviceaccounts
```

---

# 4. Install CloudWatch Observability Add-on

AWS now recommends using the CloudWatch Observability Add-on.

Install:

```bash
aws eks create-addon \
  --cluster-name <cluster-name> \
  --addon-name amazon-cloudwatch-observability
```

Verify:

```bash
aws eks list-addons --cluster-name <cluster-name>
```

Check pods:

```bash
kubectl get pods -n amazon-cloudwatch
```

---

# 5. Enable Logging for Fargate Pods

For Fargate, logging is configured using a special ConfigMap called:

```text
aws-logging
```

Create namespace:

```bash
kubectl create namespace aws-observability
```

Label namespace:

```bash
kubectl label namespace aws-observability aws-observability=enabled
```

---

# 6. Create Fluent Bit ConfigMap

Create file:

```bash
vi aws-logging.yaml
```

Add:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-logging
  namespace: aws-observability
data:
  output.conf: |
    [OUTPUT]
        Name cloudwatch_logs
        Match   *
        region <region>
        log_group_name /aws/eks/fargate-logs
        log_stream_prefix from-fluent-bit-
        auto_create_group true

  filters.conf: |
    [FILTER]
        Name kubernetes
        Match *
        Merge_Log On
        Keep_Log Off
        Buffer_Size 0

  parsers.conf: |
    [PARSER]
        Name crio
        Format Regex
        Regex ^(?<time>.+) (?<stream>stdout|stderr) (?<logtag>P|F) (?<log>.*)$
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```

Apply:

```bash
kubectl apply -f aws-logging.yaml
```

---

# 7. Verify Logging

Deploy sample app:

```bash
kubectl create deployment nginx --image=nginx
```

Check pod:

```bash
kubectl get pods
```

Generate logs:

```bash
kubectl logs deployment/nginx
```

Now go to:

* [AWS CloudWatch Console](https://console.aws.amazon.com/cloudwatch/?utm_source=chatgpt.com)
* Log groups
* `/aws/eks/fargate-logs`

You should see logs.

---

# 8. Verify Container Insights Metrics

Go to:

* CloudWatch
* Container Insights
* Performance Monitoring
* Select EKS cluster

You should see:

* CPU
* Memory
* Pod metrics
* Namespace metrics

---

# 9. Common Issues

## Logs not appearing

Check:

```bash
kubectl get configmap aws-logging -n aws-observability
```

Ensure namespace label exists:

```bash
kubectl get ns aws-observability --show-labels
```

---

## Access denied errors

Ensure Fargate pod execution role has:

```text
CloudWatchLogsFullAccess
```

or at minimum:

```text
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
```

---

# 10. Important Note for Fargate

Unlike EC2 worker nodes:

* No DaemonSets
* No direct node access
* Logging handled by AWS-managed Fluent Bit
* Only namespace-labelled workloads are monitored

---

# Useful AWS Documentation

* [EKS Fargate Logging Docs](https://docs.aws.amazon.com/eks/latest/userguide/fargate-logging.html?utm_source=chatgpt.com)
* [CloudWatch Container Insights Docs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html?utm_source=chatgpt.com)
* [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html?utm_source=chatgpt.com)
