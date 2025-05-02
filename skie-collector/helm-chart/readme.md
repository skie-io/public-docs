# SKIE Kubernetes Collector – Helm Chart Guide

This document provides instructions for installing and configuring the SKIE Kubernetes Collector using Helm.

## Prerequisites

- Helm version 3 or higher
- SKIE Customer Identifier and access token
- SKIE Endpoint

### Metrics Server Dependency

The SKIE Collector requires the Kubernetes Metrics Server. By default, the chart installs it for you. If you already have Metrics Server running in your cluster, you can skip its installation by setting the following parameter to `false`:

```bash
--set global.installMetricsServer=false
```

To check if Metrics Server is already installed:

```bash
kubectl get deployments --all-namespaces | grep metrics-server
```
Look for READY replicas > 0 in the output.

```bash
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
```
Look for `AVAILABLE True` in the output—this confirms the metrics API is registered and reachable.

If both checks pass, you do not need to install Metrics Server with the SKIE Collector.

## Installation

Install the collector using Helm:

```bash
helm upgrade --install skie-k8s-collector \
  --set global.skieToken="your-secret-token" \
  --set global.clusterName="your-cluster-name" \
  --set global.customerIdentifier="your-customer-id" \
  oci://public.ecr.aws/x7r0w8m0/skie-helm-charts/skie-k8s-collector
```

If you are using a private VPC Endpoint, set the parameter `global.skieEndpoint` and use HTTP instead of HTTPS:

```bash
helm upgrade --install --create-namespace --namespace skie-k8s-collector \
  skie-k8s-collector \
  --set global.skieEndpoint="http://your-lattice-endpoint.aws" \
  --set global.skieToken="your-secret-token" \
  --set global.clusterName="your-cluster-name" \
  --set global.customerIdentifier="your-customer-id" \
  oci://public.ecr.aws/x7r0w8m0/skie-helm-charts/skie-k8s-collector
```

## Configuration Parameters

| Parameter                   | Description                        | Required |
|-----------------------------|------------------------------------|----------|
| `global.skieEndpoint`       | SKIE platform endpoint URL         | No       |
| `global.skieToken`          | Authentication token               | Yes      |
| `global.customerIdentifier` | Your unique customer identifier    | Yes      |
| `global.clusterName`        | Your unique customer identifier    | Yes      |

## Verification

To verify the collector is running:

```bash
kubectl get pods -n skie-k8s-collector -l app=skie-k8s-collector
```

## Troubleshooting

Common issues and solutions:

1. **Invalid token:** Ensure your SKIE token is valid and properly configured.
2. **Connection issues:** Verify your cluster has outbound access to the SKIE endpoint.
3. **Pod status:** Check pod logs using:
   ```bash
   kubectl logs -n skie-k8s-collector -l app=skie-k8s-collector
   ```
4. **Metrics unavailable:** Confirm Metrics Server is running and collecting data.

## Support

For questions or issues, contact SKIE support or visit our documentation portal.
