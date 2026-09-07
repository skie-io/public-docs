# Automatic collector updates

This page describes the upcoming [collector 1.x release](readme.md). Automatic
updates default to `true` for plain Helm installations. The first upgrade from
`0.0.1` is manual and requires the [Metrics Server migration](upgrading.md).

## How updates work

The `skie-k8s-collector-updater` CronJob checks SKIE's public OCI registry every six
hours at a deterministic slot based on your namespace and cluster name, with up
to five minutes of jitter. It upgrades only an existing, deployed Helm release.
It preserves your stored overrides, applies new chart defaults, skips unchanged
or older versions, and attempts rollback if an upgrade fails readiness checks.

By default it selects stable releases within the installed major (`^1` for 1.x).
It excludes prereleases and never crosses a major version, even if you configure
a broader range. Major upgrades remain manual because they can change permissions,
resource names, or Kubernetes requirements. An explicitly stored
`autoUpdate.enabled: false` stays disabled across upgrades.

| Setting | Default | Purpose |
|---|---|---|
| `autoUpdate.enabled` | `true` | Set `false` to opt out; required for GitOps |
| `autoUpdate.versionConstraint` | Installed major, e.g. `^1` | Use `~1.0` for 1.0.x or an exact version to pin |
| `autoUpdate.schedule` | Six-hourly slot | Override with a five-field cron expression |
| `autoUpdate.timeZone` | `Etc/UTC` | Time zone for the schedule |
| `autoUpdate.jitterSeconds` | `300` | Set `0` to disable jitter |
| `autoUpdate.helmTimeout` | `20m` | Per-upgrade/rollback readiness timeout |

The combined jitter and two timeouts must leave at least five minutes within the
two-hour Job deadline. Chart validation rejects unsupported settings.

## Argo CD and Flux

Explicitly disable the updater in the collector's Helm values:

```yaml
autoUpdate:
  enabled: false
```

Argo CD renders manifests without creating Helm release storage. Flux manages its
own Helm release, and two independent upgraders would compete. Use the GitOps
controller's chart version selection and sync policy for updates instead.

For a wrapper chart, place these values under the collector dependency name or
alias. Commit an exact dependency version and `Chart.lock`, and update them through
your normal review process. Before the first 1.x sync, complete the
[GitOps migration](upgrading.md#gitops-migration).

## Network and permissions

The updater pulls charts from `public.ecr.aws:443`. Nodes must also be able to
pull the configured collector, kube-state-metrics, and updater images. PrivateLink
keeps your metrics traffic on the private endpoint; chart and image downloads
still need registry access. SKIE does not initiate an inbound connection to your
cluster to perform an update.

Use a dedicated release namespace with no unrelated privileged service accounts.
The updater can read Secrets (including the SKIE token and Helm history) and
administer chart resource types in that namespace. It can create workloads using
other service accounts there. It also holds the collectors' cluster monitoring
permissions and can maintain the three component ClusterRoles/Bindings and its
own fourth pair. Kubernetes cannot restrict ClusterRole/Binding creation by name;
its escalation checks still apply. There is no `escalate`, `bind`, or `impersonate`
grant and no direct workload write grant in `kube-system`.

The inherited `get nodes/proxy` permission permits privileged kubelet operations,
including execution in pods across namespaces. Namespace-scoped Secret reads
therefore do not guarantee isolation from other namespaces. Review the
[Kubernetes RBAC guidance](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#access-to-proxy-subresource-of-nodes)
and these permissions before installing. Set `autoUpdate.enabled: false` if you
do not want the updater's permissions or behavior.

The updater uses a digest-pinned Helm 4.2.4 image, runs as non-root with a read-only
root filesystem, and keeps temporary Helm files in ephemeral storage. Chart
publishers can deliver manifests with the updater's effective permissions; only
use a trusted registry. The initial updater does not verify chart signatures.

## Stop, audit, and recover

Use your actual release namespace in these commands. For an immediate pause:

```sh
kubectl -n skie-k8s-collector patch cronjob skie-k8s-collector-updater \
  -p '{"spec":{"suspend":true}}'
```

Suspension does not stop a Job already running, and a future Helm upgrade can
reconcile the setting. To disable permanently, upgrade the **currently installed
chart version** with `--reset-then-reuse-values --set autoUpdate.enabled=false`.
For GitOps, commit `autoUpdate.enabled: false` to your controller's values.

```sh
helm history skie-k8s-collector -n skie-k8s-collector
kubectl -n skie-k8s-collector get cronjobs,jobs
kubectl -n skie-k8s-collector logs job/JOB_NAME
kubectl -n skie-k8s-collector get configmap skie-k8s-collector-updater -o yaml
```

A non-deployed release (for example, `pending-upgrade`) stops automatic updates
and requires manual inspection and recovery. A broken updater script requires a
manual chart upgrade or repair. Avoid Helm `--debug` in updater logs because it
can print rendered Secrets.

The collectors report `skie.chart.version` and `skie.autoupdate.enabled`. Changing
only the updater setting does not restart collectors; the enrollment attribute
refreshes on their next restart or chart upgrade. Continue monitoring actual
metric receipt, even when Helm reports a successful upgrade.
