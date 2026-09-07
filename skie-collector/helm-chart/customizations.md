# SKIE Kubernetes Collector – Custom Installation Guide

This guide targets the upcoming [collector 1.x release](readme.md). It explains
how to provide externally managed configuration and Secrets. Argo CD, Flux, or
CI-managed releases should explicitly set `autoUpdate.enabled: false` so one
system controls upgrades.

## External Secret

You can manage only the Secret externally and leave the ConfigMap chart-managed.
This lets Helm keep endpoint and inventory variables current during upgrades.
Create the following Secret through your secret-management workflow in the
**same namespace as the collector release**:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: skie-k8s-collector-secret
  namespace: skie-k8s-collector
type: Opaque
stringData:
  bearertokenauth: YOUR_TOKEN
```

Set `global.createSecret: false` and omit `global.skieToken` from your Helm values.
Use the exact Secret name above. Do not commit a real token to source control.

## External ConfigMap

Only disable `global.createConfigMap` if your automation maintains all the
following variables. This example uses manual/GitOps updates:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: skie-k8s-collector-cm
  namespace: skie-k8s-collector
data:
  K8S_CLUSTER_NAME: YOUR_CLUSTER_NAME
  SKIE_FULL_ENDPOINT: https://app.skie.io/api/v1/organizations/YOUR_CUSTOMER_ID/kubernetes_metrics
  CUSTOMER_IDENTIFIER: YOUR_CUSTOMER_ID
  PROVIDER: aws
  SKIE_CHART_VERSION: "1.0.0"
  SKIE_AUTOUPDATE_ENABLED: "false"
```

The ConfigMap name is fixed. `SKIE_FULL_ENDPOINT` must be a complete URL, including
the organization path; raw Helm template expressions in an external ConfigMap are
not evaluated. For a private endpoint, use
`http://YOUR_PRIVATE_DNS:3000/api/v1/organizations/YOUR_CUSTOMER_ID/kubernetes_metrics`.

Update `SKIE_CHART_VERSION` on every chart upgrade and keep
`SKIE_AUTOUPDATE_ENABLED` aligned with the release's actual setting. The in-cluster
updater does not maintain externally managed objects. Keep the ConfigMap
chart-managed when using automatic updates unless your own automation handles
these changes.

## Install with both objects externally managed

Create the namespace and both objects before installing. After `1.0.0` is
published, run:

```sh
helm upgrade --install skie-k8s-collector \
  oci://public.ecr.aws/x7r0w8m0/skie-helm-charts/skie-k8s-collector \
  --namespace skie-k8s-collector --create-namespace --version 1.0.0 \
  --set global.clusterName=YOUR_CLUSTER_NAME \
  --set global.customerIdentifier=YOUR_CUSTOMER_ID \
  --set global.createConfigMap=false --set global.createSecret=false \
  --set autoUpdate.enabled=false
```

For an existing `0.0.1` release, complete the [migration](upgrading.md) first.
When only the Secret is external, leave `global.createConfigMap` at its default
`true` and configure your endpoint and cluster identity through Helm values.

## Apply configuration or token changes

External ConfigMap and Secret updates do not automatically restart the collectors.
After updating either object, restart both workloads through your deployment
workflow, or use:

```sh
kubectl -n skie-k8s-collector rollout restart \
  daemonset/skie-k8s-collector-opentelemetry-collector-agent
kubectl -n skie-k8s-collector rollout restart \
  deployment/skie-k8s-collector-opentelemetry-collector-deployment
```

Confirm rollout completion and receipt of metrics in SKIE. See
[automatic updates](automatic-updates.md) for updater settings and audit commands.
