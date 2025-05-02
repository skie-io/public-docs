# SKIE Kubernetes Collector – Custom Installation Guide

This guide explains how to install the SKIE Collector using your own CI/CD tooling, allowing you to customize parameters and manage secrets according to your environment.

## Custom ConfigMap and Secret Creation

The SKIE Collector expects a ConfigMap containing your cluster information and a Secret containing your authentication token. You can create these resources yourself and instruct Helm not to create them automatically.

### 1. Creating the ConfigMap

Create a ConfigMap named `skie-k8s-collector-cm` with the following structure:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: skie-k8s-collector-cm
data:
  K8S_CLUSTER_NAME: your-cluster-name
  SKIE_FULL_ENDPOINT: {{ printf "%s/api/v1/organizations/%s/kubernetes_metrics" .Values.global.skieEndpoint your-customer-id | quote }}
  CUSTOMER_IDENTIFIER: your-customer-id
  PROVIDER: aws
```

- `SKIE_FULL_ENDPOINT` is a dynamically created URL that combines your SKIE endpoint and customer identifier. For example:
  ```
  https://zee.skie.io/api/v1/organizations/asbs7d0c-6409-477f-bs12-facg4dmb5ab4/kubernetes_metrics
  ```

### 2. Creating the Secret

Create a Secret named `skie-k8s-collector-secret` to store your authentication token:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: skie-k8s-collector-secret
type: Opaque
stringData:
  bearertokenauth: your-token
```

> **Note:** Both the ConfigMap and Secret must use the exact names above, as the SKIE Collector expects these resource names.

### 3. Disabling Automatic Resource Creation in Helm

Tell Helm not to create the ConfigMap or Secret by setting the following parameters:

```bash
--set global.createConfigMap=false --set global.createSecret=false
```

### 4. Example Helm Installation Command

```bash
helm upgrade --install skie-k8s-collector \
  --set global.createConfigMap=false \
  --set global.createSecret=false \
  oci://public.ecr.aws/x7r0w8m0/skie-helm-charts/skie-k8s-collector
```

## Support

For issues or questions, contact SKIE support or visit our documentation portal.
