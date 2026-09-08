# SKIE Kubernetes Collector – Helm Chart Guide

This guide targets collector **1.0.0 and later**. Prerelease versions such as
`1.0.0-rc.5` are for SKIE's own canary validation and are not selected by the
default update constraint. Existing `0.0.1` installations must follow the
[migration guide](upgrading.md) before upgrading.

**Automatic updates are enabled by default.** Plain Helm installations receive
compatible releases within the installed major version. Argo CD and Flux users
must explicitly set `autoUpdate.enabled: false` and manage upgrades through their
controller. See [automatic updates](automatic-updates.md) for controls and permissions.

## Prerequisites

- Kubernetes 1.27 or newer and Helm 3.14+ or Helm 4.
- Your SKIE customer identifier, access token, and cluster name.
- Permission to create the chart's namespace resources and cluster RBAC objects.
- Access to your SKIE metrics endpoint. The default is `https://app.skie.io`.
- Outbound HTTPS to `public.ecr.aws:443` for automatic chart downloads, plus access
  to the configured container image registries. A private metrics endpoint does
  not provide access to those registries.

## Metrics Server

Collector 1.x no longer installs or owns Metrics Server, and
`global.installMetricsServer` is ignored. Keep your existing Metrics Server for
HPA and `kubectl top`; the collector's current receivers do not query
`metrics.k8s.io` directly.

Check an existing installation with:

```sh
kubectl -n kube-system get deployment metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl top nodes
```

If your cluster needs Metrics Server and does not already have it, install the
separate infrastructure chart:

```sh
helm install skie-metrics-server \
  oci://public.ecr.aws/x7r0w8m0/skie-helm-charts/skie-metrics-server \
  --namespace kube-system --version 1.0.0
```

Its nine objects are retained on uninstall. For Metrics Server previously
installed by collector `0.0.1`, use the [migration guide](upgrading.md) instead.

## Install with Helm

Use the fixed release name `skie-k8s-collector` and a dedicated namespace such as
`skie-k8s-collector`. The updater rejects `default`, `kube-system`, `kube-public`,
and `kube-node-lease`. For an existing release, keep its actual namespace and
follow the migration instructions.

Save your customer overrides in a protected `skie-values.yaml` file. Keep the
token out of source control; [external Secrets](customizations.md) are also supported.

```yaml
global:
  skieToken: YOUR_TOKEN
  customerIdentifier: YOUR_CUSTOMER_ID
  clusterName: YOUR_CLUSTER_NAME
  provider: aws
```

```sh
helm upgrade --install skie-k8s-collector \
  oci://public.ecr.aws/x7r0w8m0/skie-helm-charts/skie-k8s-collector \
  --namespace skie-k8s-collector --create-namespace --version 1.0.0 \
  --values skie-values.yaml
```

No updater opt-in flag is needed. To disable automatic updates, add this to your
values file before installing (required for Argo CD and Flux):

```yaml
autoUpdate:
  enabled: false
```

Store only your overrides, not a full copy of chart defaults. Automatic upgrades
preserve stored overrides while adopting new defaults; copied defaults would pin
settings such as collector images and processors.

For a private VPC endpoint, set the base endpoint in your values file using HTTP
and port 3000:

```yaml
global:
  skieEndpoint: http://YOUR_PRIVATE_DNS:3000
```

Merge this setting into the existing `global` block. The chart appends your
organization's metrics path. See the [CloudFormation guide](../cloudformation/readme.md)
for obtaining the private DNS name.

## Configuration

| Parameter | Default / purpose |
|---|---|
| `global.skieEndpoint` | `https://app.skie.io`; public or private base URL |
| `global.skieToken` | Your access token, unless using an external Secret |
| `global.customerIdentifier` | Your SKIE customer identifier |
| `global.clusterName` | A stable name identifying this Kubernetes cluster |
| `global.provider` | `aws`; cloud provider attribute |
| `global.createSecret`, `global.createConfigMap` | `true`; see [customizations](customizations.md) before disabling |
| `global.chartVersion` | Chart-owned inventory marker; do not override |
| `autoUpdate.enabled` | `true`; explicitly set `false` for GitOps or manual updates |

## Verify and troubleshoot

```sh
kubectl -n skie-k8s-collector get pods -l app.kubernetes.io/instance=skie-k8s-collector
kubectl -n skie-k8s-collector logs -l app.kubernetes.io/instance=skie-k8s-collector --all-containers=true --tail=100
kubectl -n skie-k8s-collector get cronjob skie-k8s-collector-updater
helm history skie-k8s-collector -n skie-k8s-collector
```

The CronJob is absent when automatic updates are disabled. Check token validity,
endpoint connectivity, and collector logs if metrics do not arrive. Confirm data
receipt in SKIE as well as pod readiness; Helm cannot detect a Ready collector
that exports no data. For update failures, see [recovery](automatic-updates.md#stop-audit-and-recover).

Contact SKIE support for deployment assistance.
