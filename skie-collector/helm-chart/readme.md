# SKIE Kubernetes Collector – Helm Chart Guide

Install the SKIE collector with Helm. Two optional pieces sit alongside it: the
separate Metrics Server chart, and automatic updates, which are enabled by
default.

Already running collector `0.0.1`? Follow the [migration guide](upgrading.md)
instead — the first upgrade is manual.

## Prerequisites

- Kubernetes 1.27 or newer and Helm 3.14+ or Helm 4.
- Your SKIE customer identifier, access token, and cluster name.
- Permission to create the chart's namespace resources and cluster RBAC objects.
- Access to your SKIE metrics endpoint. The default is `https://app.skie.io`.
- Outbound HTTPS to `public.ecr.aws:443` for automatic chart downloads, plus access
  to the configured container image registries. A private metrics endpoint does
  not provide access to those registries.

## Metrics Server

**The collector does not need it.** Nothing the collector reports comes from
Metrics Server: it reads usage from the kubelet directly, object inventory from
the API server, and the rest from kube-state-metrics. The HPA series it collects
are specification and info metrics, which are present whether or not Metrics
Server is running. If your cluster has none, the collector still reports
everything it normally does.

Metrics Server matters for **your** cluster, not for SKIE: without it `kubectl
top` returns "Metrics API not available" and any HorizontalPodAutoscaler reports
`<unknown>` targets and will not scale.

Check whether you already have one:

```sh
kubectl -n kube-system get deployment metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl top nodes
```

If those commands fail and you want `kubectl top` and working autoscaling,
install the separate chart. It is independent of the collector — install it
before or after, in either order:

```sh
helm install skie-metrics-server \
  oci://public.ecr.aws/x7r0w8m0/skie-helm-charts/skie-metrics-server \
  --namespace kube-system --version '^1.0.0'
```

Its nine objects carry `helm.sh/resource-policy: keep`, so uninstalling the chart
leaves Metrics Server running and your autoscaling intact; delete the objects
yourself if you really want it gone.

If your cluster already has a Metrics Server, do not install this chart on top of
it. Both would claim the same `v1beta1.metrics.k8s.io` API service.

## Install with Helm

Use the fixed release name `skie-k8s-collector` and a dedicated namespace such as
`skie-k8s-collector`. The updater rejects `default`, `kube-system`, `kube-public`,
and `kube-node-lease`.

Save your customer overrides in a protected `skie-values.yaml` file, and keep the
token out of source control:

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
  --namespace skie-k8s-collector --create-namespace --version '^1.0.0' \
  --values skie-values.yaml
```

Store only your overrides, not a full copy of chart defaults. Upgrades preserve
stored overrides while adopting new defaults; copied defaults would pin settings
such as collector images and processors.

For a private VPC endpoint, set the base endpoint in your values file using HTTP
and port 3000:

```yaml
global:
  skieEndpoint: http://YOUR_PRIVATE_DNS:3000
```

Merge this setting into the existing `global` block. The chart appends your
organization's metrics path. See the [CloudFormation guide](../cloudformation/readme.md)
for obtaining the private DNS name.

## Automatic updates

A `skie-k8s-collector-updater` CronJob checks once a day for a newer chart and
upgrades the release in place. It stays within the installed major version,
preserves your stored overrides, and rolls back an upgrade that fails its
readiness checks. Major upgrades stay manual.

To opt out, add this to your values file before installing:

```yaml
autoUpdate:
  enabled: false
```

Set the same flag if Argo CD or Flux manages the release, so only your controller
performs upgrades.

To pause updates on a running release without changing your values:

```sh
kubectl -n skie-k8s-collector patch cronjob skie-k8s-collector-updater \
  -p '{"spec":{"suspend":true}}'
```

## Configuration

| Parameter | Default / purpose |
|---|---|
| `global.skieEndpoint` | `https://app.skie.io`; public or private base URL |
| `global.skieToken` | Your access token |
| `global.customerIdentifier` | Your SKIE customer identifier |
| `global.clusterName` | A stable name identifying this Kubernetes cluster |
| `global.provider` | `aws`; cloud provider attribute |
| `global.chartVersion` | Chart-owned inventory marker; do not override |
| `autoUpdate.enabled` | `true`; set `false` to manage upgrades yourself |
| `autoUpdate.versionConstraint` | Installed major, e.g. `^1`; use `~1.0` or an exact version to pin |
| `autoUpdate.schedule` | Daily slot; override with a five-field cron expression |

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
that exports no data.

Contact SKIE support for deployment assistance.
