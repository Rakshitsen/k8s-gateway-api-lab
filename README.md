# k8s-gateway-api-lab

Hands-on lab for migrating Kubernetes ingress from classic Ingress controllers (Nginx/ALB) to the **Gateway API** (Envoy) on both **AWS EKS** and **GCP GKE**. Includes full setup guides, working manifests, and a side-by-side Ingress vs Gateway API comparison for the same two apps (Vault, Jenkins).

## What's in here

| Path | Purpose |
|---|---|
| `EKS-Setup-Complete-Guide.md` | End-to-end AWS EKS setup: cluster creation, Nginx Ingress, AWS Load Balancer Controller, EBS storage, Vault deployment |
| `GKE-Setup-Complete-Guide.md` | End-to-end GCP GKE setup: private cluster + VPC, jump host access, Nginx Ingress, migration to Envoy Gateway API, NEG-backed load balancing |
| `ingress-setup-files/` | Classic Ingress manifests (controller + app ingress rules) for both clouds |
| `gateway-setup-files/` | Gateway API manifests (GatewayClass, Gateway, HTTPRoutes, Envoy proxy config) for both clouds |

```
k8s-gateway-api-lab/
├── EKS-Setup-Complete-Guide.md
├── GKE-Setup-Complete-Guide.md
├── ingress-setup-files/
│   ├── ingress-controller-aws.yaml
│   ├── ingress-controller-gcp.yaml
│   ├── ingress-nginx-tgb.yaml
│   ├── jenkins-ingress.yaml
│   └── vault-ingress.yaml
└── gateway-setup-files/
    ├── envoy-proxy.yaml
    ├── envoy-proxy-gcp.yaml
    ├── gateway-class.yaml
    ├── gateway.yaml
    ├── gateway-envoy-tgb.yaml
    ├── jenkins-route.yaml
    └── vault-route.yaml
```

## Why this lab exists

Ingress has been the default way to expose Kubernetes services for years, but it's limited: no native support for multiple protocols, weak traffic-splitting, and every cloud/controller extends it differently via annotations. Gateway API fixes this with a standard, role-oriented model (`GatewayClass` → `Gateway` → `HTTPRoute`) that's portable across implementations.

This repo documents doing that migration for real, on two clouds, with two real apps:

- **Vault** and **Jenkins** deployed behind both an Ingress controller and a Gateway API stack (Envoy), so you can compare configuration and behavior directly.
- **AWS path:** EKS → Nginx Ingress + ALB → migrate to Envoy Gateway with target group bindings.
- **GCP path:** Private GKE → Nginx Ingress + external LB → migrate to Envoy Gateway with NEGs.

## Quick start

1. Pick your cloud and follow the matching guide:
   - AWS: [`EKS-Setup-Complete-Guide.md`](./EKS-Setup-Complete-Guide.md)
   - GCP: [`GKE-Setup-Complete-Guide.md`](./GKE-Setup-Complete-Guide.md)
2. Each guide is self-contained — prerequisites, cluster creation, Ingress setup, then Gateway API migration, in order.
3. Apply manifests from `ingress-setup-files/` or `gateway-setup-files/` as directed in the guide (paths in the guides currently reference raw YAML inline — apply the corresponding file from these folders instead).

## Prerequisites (both paths)

- `kubectl`, `helm` installed
- A registered domain for DNS records
- Cloud CLI configured:
  - AWS: `eksctl`, AWS CLI with credentials
  - GCP: `gcloud`, an existing VPC/subnet/NAT setup

## Stack covered

- **AWS:** EKS, Nginx Ingress, AWS Load Balancer Controller, EBS CSI, Envoy Gateway
- **GCP:** GKE (private cluster), IAP jump host, Nginx Ingress, Google Cloud external LB, NEGs, Envoy Gateway
- **Gateway API:** GatewayClass, Gateway, HTTPRoute (Envoy Gateway implementation)
- **Apps:** HashiCorp Vault, Jenkins

## References

- [Kubernetes Gateway API](https://gateway.api.k8s.io/)
- [Envoy Gateway](https://gateway.envoyproxy.io/)
- [EKS Documentation](https://docs.aws.amazon.com/eks/)
- [GKE Documentation](https://cloud.google.com/kubernetes-engine/docs)

## Author

Rakshit Sen ([@Rakshitsen](https://github.com/Rakshitsen))
