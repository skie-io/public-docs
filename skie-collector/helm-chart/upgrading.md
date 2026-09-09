# Upgrade from collector 0.0.1 to 1.x

Run this once with a cluster administrator and Helm 3.14+ or Helm 4. Version
`0.0.1` remains frozen; the first major upgrade is manual.

## Prepare your release

Find the existing release and use its actual namespace in every command:

```sh
helm list --all-namespaces --filter '^skie-k8s-collector$'
```

The examples below use `skie-k8s-collector`. Older public-endpoint instructions
installed into `default` when no namespace was supplied. Changing `--namespace`
does not move a release. For a release in `default` or a Kubernetes system
namespace, retain that namespace and add `--set autoUpdate.enabled=false` to the
upgrade command. Automatic updates require a dedicated namespace.

Securely back up customer overrides with `helm get values skie-k8s-collector -n
YOUR_NAMESPACE -o yaml`. The output can contain your token. Keep only intentional
customer overrides, remove any `global.chartVersion` override, and avoid retaining
full copies of old image/processor defaults.

## Retain Metrics Server first

If `0.0.1` installed Metrics Server (`global.installMetricsServer` defaulted to
`true`), preserve all nine objects before upgrading. If you used a separately
managed Metrics Server with that flag disabled, skip this section.

Confirm ownership using your release manifest (`helm get manifest`) and the
objects' Helm release annotations. Do not annotate an unrelated installation.
For a SKIE-owned Metrics Server, all commands below must succeed:

```sh
kubectl -n kube-system annotate --overwrite \
  deployment/metrics-server service/metrics-server \
  serviceaccount/metrics-server rolebinding/metrics-server-auth-reader \
  helm.sh/resource-policy=keep
kubectl annotate --overwrite \
  clusterrole/system:metrics-server clusterrole/system:aggregated-metrics-reader \
  clusterrolebinding/system:metrics-server clusterrolebinding/metrics-server:system:auth-delegator \
  apiservice/v1beta1.metrics.k8s.io helm.sh/resource-policy=keep
```

Verify `helm.sh/resource-policy: keep` on all nine live objects before proceeding.
Skipping retention can delete Metrics Server and disrupt HPA during the upgrade.

## Upgrade with plain Helm

```sh
helm upgrade skie-k8s-collector \
  oci://public.ecr.aws/x7r0w8m0/skie-helm-charts/skie-k8s-collector \
  --namespace skie-k8s-collector --version '^1.0.0' --reset-then-reuse-values
```

This enables automatic updates unless your stored values explicitly disable them.
Add `--set autoUpdate.enabled=false` to opt out, or for releases in `default` or
system namespaces. The old `global.installMetricsServer` value is ignored in 1.x.

For an older Helm client, update Helm first. Alternatively, export only customer
overrides to a protected values file and use `--reset-values --values FILE` in
place of `--reset-then-reuse-values`.

Retained Metrics Server objects keep running outside the collector release. You
may leave them in place or arrange explicit adoption by your infrastructure
manager or the separate `skie-metrics-server` chart. Retention annotations do not
transfer ownership. Do not roll the collector back to `0.0.1` after adopting those
objects into another release.

## Verify the migration

```sh
kubectl -n skie-k8s-collector get pods -l app.kubernetes.io/instance=skie-k8s-collector
kubectl -n kube-system get deployment metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl top nodes
kubectl get hpa --all-namespaces
```

Confirm the collectors are ready, Metrics Server and any HPAs remain healthy, and
SKIE receives metrics with a 1.x `skie.chart.version`. If automatic updates are
enabled, also verify the `skie-k8s-collector-updater` CronJob in your release
namespace. See the [Helm chart guide](readme.md#automatic-updates) for the pause
command.
