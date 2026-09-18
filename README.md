# Kubernetes Gateway API on AWS EKS Lab

A guide to setting up ingress and Gateway API controllers on AWS EKS with practical examples using Vault and Jenkins.

## What's Inside

This lab covers two approaches to expose applications on Kubernetes:

- **Nginx Ingress (Traditional):** Legacy but stable; uses `Ingress` API
- **Envoy Gateway (Modern):** Standards-based; uses `Gateway` and `HTTPRoute` APIs

Both approaches integrate with AWS ALB for external traffic routing and work with the same EKS cluster—choose one or migrate between them.

## Quick Start

### 1. Set Up EKS Cluster

```bash
# Read the guide
cat EKS-Setup-Complete-Guide.md

# Follow Steps 1–6 to create cluster and install dependencies
```

### 2. Choose Your Ingress Approach

**Option A: Nginx Ingress Controller** → Use files in `ingress-setup-files/`  
Fast to deploy; widely used in production.

**Option B: Envoy Gateway (Gateway API)** → Use files in `gateway-setup-files/`  
Modern alternative; better routing semantics and observability.

### 3. Deploy Your Approach

Both are covered in `EKS-Setup-Complete-Guide.md`. Nginx is in Steps 2–4; Envoy is in the separate guide.

## File Structure

```
├── EKS-Setup-Complete-Guide.md     # Core guide (Steps 1–7)
│
├── gateway-setup-files/             # Gateway API (Envoy)
│   ├── envoy-proxy.yaml            # Envoy controller config
│   ├── gateway-class.yaml          # GatewayClass definition
│   ├── gateway.yaml                # Gateway (data plane)
│   ├── gateway-envoy-tgb.yaml      # Target group binding
│   ├── vault-route.yaml            # HTTPRoute for Vault
│   └── jenkins-route.yaml          # HTTPRoute for Jenkins
│
└── ingress-setup-files/             # Nginx Ingress (Traditional)
    ├── ingress-controller-aws.yaml # Nginx controller manifest
    ├── ingress-controller-gcp.yaml # Nginx for GCP (reference)
    ├── ingress-nginx-tgb.yaml      # Target group binding
    ├── vault-ingress.yaml          # Ingress for Vault
    └── jenkins-ingress.yaml        # Ingress for Jenkins
```

## Prerequisites

- AWS account with programmatic access
- `eksctl`, `kubectl`, `helm`, AWS CLI installed
- SSH key pair for node access
- Domain name (for DNS records)
- Basic Kubernetes knowledge

## Common Tasks

### Deploy Nginx Ingress

```bash
kubectl apply -f ingress-setup-files/ingress-controller-aws.yaml
kubectl apply -f ingress-setup-files/ingress-nginx-tgb.yaml
kubectl apply -f ingress-setup-files/vault-ingress.yaml
kubectl apply -f ingress-setup-files/jenkins-ingress.yaml
```

### Deploy Envoy Gateway

```bash
helm repo add envoy https://gateway.envoyproxy.io && helm repo update envoy
helm install eg envoy/gateway --version v1.9.1 -n envoy-gateway-system --create-namespace

kubectl apply -f gateway-setup-files/envoy-proxy.yaml
kubectl apply -f gateway-setup-files/gateway-class.yaml
kubectl apply -f gateway-setup-files/gateway.yaml
kubectl apply -f gateway-setup-files/gateway-envoy-tgb.yaml
kubectl apply -f gateway-setup-files/vault-route.yaml
kubectl apply -f gateway-setup-files/jenkins-route.yaml
```

### Migrate from Nginx to Envoy

1. Deploy Envoy Gateway (steps above)
2. Update ALB listener to point to Envoy target group
3. Verify all routes work: `kubectl get httproute -A`
4. (Optional) Remove Nginx: `helm uninstall aws-load-balancer-controller -n kube-system`

## Verification

After deployment, verify with:

```bash
# Cluster ready
kubectl get nodes

# Ingress controller / Gateway running
kubectl get pods -n ingress-nginx            # For Nginx
kubectl get pods -n envoy-gateway-system     # For Envoy

# ALB health in AWS Console
aws elbv2 describe-target-health --target-group-arn <arn>

# Applications accessible
curl -k https://vault.rakops.in
curl -k https://jenkins.rakops.in
```

## Key Differences

| Aspect | Nginx Ingress | Envoy Gateway |
|--------|---------------|---------------|
| API | Kubernetes `Ingress` | Kubernetes `Gateway` + `HTTPRoute` |
| Status | Stable, widely used | Growing adoption, standardized |
| Features | Routing, basic auth | Routing, traffic splitting, transformation |
| Learning | Easier entry | Steeper but more expressive |
| Migration | One-way (Nginx → Envoy) | Forward-compatible with Gateway API |

## Troubleshooting

**Targets showing "unhealthy" in ALB?**
- Check security group rules (health check ports: 10254 for Nginx, 19003 for Envoy)
- Verify target group health check path is correct
- Run `kubectl logs -n <namespace> <pod>` for pod logs

**Applications not responding after deployment?**
- Confirm Ingress/HTTPRoute is bound: `kubectl get ingress -A` or `kubectl get httproute -A`
- Check DNS: `nslookup vault.rakops.in` (should resolve to ALB)
- Test ALB directly: `curl -k https://<alb-dns>`

**Unsure which approach to pick?**
- **Nginx:** If your team uses Ingress already and you want stability
- **Envoy:** If you're starting fresh or want modern Gateway API features

## Next Steps

- Read `EKS-Setup-Complete-Guide.md` for detailed step-by-step instructions
- Customize the YAML files for your domain and cluster setup
- Set up monitoring/logging (beyond this lab scope)
- Explore traffic policies with Envoy (rate limiting, retries, timeouts)

## Notes

- All commands assume `ap-south-1` region; adjust for your region
- Replace placeholder values (VPC ID, domain, account ID) before applying manifests
- Save Vault unseal keys securely—do not commit to version control

---

**Lab Created By:** Rakshit  
**Last Updated:** Sep 2026
