# SKIE Kubernetes Collector

A Kubernetes collector that sends telemetry data to SKIE's observability platform.

## Prerequisites

- VPC Endpoint private
- Helm 3.x
- [Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server) installed
- Valid SKIE account and access token

### Installing Metrics Server

If metrics-server is not installed:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify metrics-server is running:
```bash
kubectl get deployment metrics-server -n kube-system
```

## Installation

Install the collector using Helm:

```bash
helm upgrade --install skie-k8s-collector \
  --set global.skieEndpoint="https://your-skie-endpoint.skie.io" \
  --set global.skieToken="your-token" \
  --set customerIdentifier="your-customer-id" \
  oci://public.ecr.aws/x7r0w8m0/skie-k8s-collector
```

### Configuration Parameters

| Parameter | Description | Required |
|-----------|-------------|----------|
| `global.skieEndpoint` | SKIE platform endpoint URL | Yes |
| `global.skieToken` | Authentication token | Yes |
| `customerIdentifier` | Your unique customer identifier | Yes |

## Verification

Check if the collector is running:

```bash
kubectl get pods -l app=skie-k8s-collector
```

## Troubleshooting

Common issues:

1. Invalid token: Check if your SKIE token is valid and properly configured
2. Connection issues: Ensure your cluster has outbound access to the SKIE endpoint
3. Pod status: Check pod logs using `kubectl logs -l app=skie-k8s-collector`
4. Metrics unavailable: Verify metrics-server is running and collecting data

## Support

For issues or questions, contact SKIE support or visit our documentation portal.

## License

Copyright (c) 2025 SKIE. All rights reserved.
