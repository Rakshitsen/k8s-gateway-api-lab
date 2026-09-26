# GCP GKE Setup & Kubernetes Ingress Configuration

## A Complete Guide for Private GKE Clusters with External Load Balancing

---

## Overview

This guide walks through setting up a Kubernetes cluster on Google Cloud Platform (GCP) using Google Kubernetes Engine (GKE) with a private control plane and nodes, secured networking, and external traffic routing via Google Cloud Load Balancer. By the end, you will have:

- A private GKE cluster with custom VPC and subnets
- Private nodes with restricted internet access (via NAT)
- Secure cluster access via IAP (Identity-Aware Proxy) jump host
- Nginx Ingress Controller routing external traffic
- Google Cloud external load balancer with health checks
- Network Endpoint Groups (NEG) for dynamic pod discovery
- DNS configuration pointing to your applications

**Region Used:** asia-south1 (Mumbai)  
**Lab Cluster Name:** rakops-cluster

---

## Prerequisites

Before starting, ensure you have:

- Google Cloud project with billing enabled
- `gcloud` CLI installed and authenticated (`gcloud auth login`)
- `kubectl` installed
- `gsutil` installed (comes with gcloud CLI)
- Basic understanding of GCP networking (VPCs, subnets, firewall rules)
- Basic understanding of Kubernetes
- Domain registered (for DNS records)
- Existing VPC, subnets, router, and NAT gateway (see [VPC Creation Guide](https://github.com/Rakshitsen/kubernetes-ha-gcp/blob/main/guide/01-create-vpc.md))

---

## Step 1: Create VPC & Network Infrastructure

Your GKE cluster requires a custom VPC with properly configured subnets, router, and NAT. This step is prerequisite but detailed separately.

### Prerequisite: External VPC Setup

Follow the [VPC Creation Guide](https://github.com/Rakshitsen/kubernetes-ha-gcp/blob/main/guide/01-create-vpc.md) to set up:

- **VPC:** `rakops-vpc-dev` (or your preferred name)
- **Subnet:** `mumbai-subnet` in region `asia-south1` with:
  - Primary IP range: `10.0.0.0/20` (for nodes)
  - Secondary range for pods: `10.4.0.0/14`
  - Secondary range for services: `10.0.16.0/20`
- **Router:** For routing outbound traffic
- **NAT Gateway:** For private nodes to access external resources (Google APIs, Docker Hub, etc.)
- **Firewall Rules:** Allowing traffic between nodes and load balancer health checks

**Why separate?** VPC setup is complex and reusable across multiple clusters. Once created, you can launch multiple GKE clusters in the same VPC.

---

## Step 2: Create Private GKE Cluster

Create a private GKE cluster with custom networking and security hardening.

### Choose Your Approach

**Option A: Simple (Default IP Ranges)**
Use this for quick labs or testing. GCP auto-assigns secondary ranges.

```bash
gcloud container clusters create rakops-cluster \
  --region=asia-south1 \
  --num-nodes=1 \
  --spot \
  --machine-type=e2-medium \
  --disk-type=pd-standard \
  --disk-size=25 \
  --network=rakops-vpc-dev \
  --subnetwork=mumbai-subnet \
  --enable-ip-alias \
  --enable-private-nodes \
  --enable-private-endpoint \
  --no-enable-cloud-logging \
  --no-enable-cloud-monitoring \
  --project=YOUR_PROJECT_ID
```

**Option B: Advanced (Custom Secondary Ranges)**
Use this for production or when you need fine-grained IP control. Requires pre-defined secondary ranges in your subnet.

```bash
gcloud container clusters create rakops-cluster \
  --region=asia-south1 \
  --num-nodes=1 \
  --spot \
  --machine-type=e2-medium \
  --disk-type=pd-standard \
  --disk-size=25 \
  --network=rakops-vpc-dev \
  --subnetwork=mumbai-subnet \
  --cluster-secondary-range-name=pods \
  --services-secondary-range-name=services \
  --enable-ip-alias \
  --enable-private-nodes \
  --enable-private-endpoint \
  --no-enable-cloud-logging \
  --no-enable-cloud-monitoring \
  --project=YOUR_PROJECT_ID
```

### Which Option to Use?

| Feature | Option A (Simple) | Option B (Advanced) |
|---------|-------------------|-------------------|
| Setup time | 2 minutes | 5 minutes (includes VPC prep) |
| IP range flexibility | None (auto-assigned) | Full control (pre-define ranges) |
| Multiple clusters | Harder (IP conflicts) | Easy (each cluster gets own ranges) |
| Best for | Labs, testing | Production, multi-cluster setups |
| Prerequisite | None | Custom VPC with secondary ranges defined |

**Recommendation:** Use **Option A** if this is your first cluster. Use **Option B** if following the [VPC Creation Guide](https://github.com/Rakshitsen/kubernetes-ha-gcp/blob/main/guide/01-create-vpc.md) (which creates secondary ranges).

This guide assumes **Option B** (custom ranges). If using Option A, the rest of the guide applies—just skip the `--cluster-secondary-range-name` and `--services-secondary-range-name` flags.

### Parameter Breakdown

**Common Flags (Both Options):**

| Flag | Purpose |
|------|---------|
| `--region=asia-south1` | Region  |
| `--num-nodes=1` | Start with 1 node per zone|
| `--spot` | Use spot VMs to save 70% on compute costs |
| `--machine-type=e2-medium` | 2 vCPU, 4 GB RAM (sufficient for testing) |
| `--network=rakops-vpc-dev` | VPC network to use |
| `--subnetwork=mumbai-subnet` | Subnet within the VPC |
| `--enable-ip-alias` | Use VPC-native networking (pod IP routing within VPC) |
| `--enable-private-nodes` | Nodes get no public IPs (secure) |
| `--enable-private-endpoint` | Control plane is private (requires jump host access) |
| `--no-enable-cloud-logging` | Disable logs to reduce costs (enable in production) |
| `--no-enable-cloud-monitoring` | Disable metrics to reduce costs (enable in production) |

**Option B Only (Custom Secondary Ranges):**

| Flag | Purpose |
|------|---------|
| `--cluster-secondary-range-name=pods` | Use this pre-defined secondary range for pod IPs (e.g., 10.4.0.0/14) |
| `--services-secondary-range-name=services` | Use this pre-defined secondary range for service IPs (e.g., 10.0.16.0/20) |

**Why Custom Ranges Matter?**
- **Option A:** GCP assigns ranges automatically (e.g., 10.4.0.0/14 for pods). Works but less control.
- **Option B:** You pre-define ranges in the subnet. Better for multi-cluster setups, CIDR planning, and avoiding IP conflicts.

### What Happens

The cluster takes **5-10 minutes** to create. GCP will:
1. Provision 1 private node in the subnet
2. Create control plane with private endpoint
3. Set up cluster networking (CNI, service networking)
4. Install system add-ons (CoreDNS, kube-proxy, etc.)

### Verify Cluster Created

Once complete, list your clusters:

```bash
gcloud container clusters list --region=asia-south1
```

Expected output:

```
NAME              LOCATION       MASTER_VERSION   MASTER_IP  STATUS
rakops-cluster    asia-south1    1.30.x           PRIVATE    RUNNING
```

---

## Step 3: Create Jump Host for Cluster Access

Since the GKE cluster has a **private control plane and nodes**, you cannot access it from your local machine directly. You need a **jump host** (bastion) in the same VPC to access the cluster via SSH tunneling.

### Why Jump Host?

- Control plane endpoint is private (no public IP)
- Nodes have no public IPs
- Jump host sits in the same VPC and can reach both
- Access is secured via IAP (Identity-Aware Proxy) — no port 22 exposed to internet

### Create Jump Host VM

```bash
gcloud compute instances create jump-host \
  --zone=asia-south1-a \
  --machine-type=e2-small \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --image-family=ubuntu-2404-lts-amd64 \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=10GB \
  --boot-disk-type=pd-standard \
  --network=rakops-vpc-dev \
  --subnet=mumbai-subnet \
  --no-address \
  --service-account=jump-host-cluster-access@YOUR_PROJECT_ID.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --project=YOUR_PROJECT_ID
```

### Parameter Breakdown

| Flag | Purpose |
|------|---------|
| `--zone=asia-south1-a` | Zone (must match cluster region) |
| `--machine-type=e2-small` | Smallest VM (1 vCPU, 2 GB RAM) — sufficient |
| `--no-address` | No public IP (access only via IAP) |
| `--service-account=...` | Service account with GKE cluster access permissions |
| `--scopes=cloud-platform` | Full GCP API access from the VM |

### Prerequisites: IAM Role & Firewall

Before creating the jump host, ensure:

1. **Service Account** with role `	roles/container.admin` is created:
   ```bash
   gcloud iam service-accounts create jump-host-cluster-access --display-name="Jump Host"
   gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
     --member="serviceAccount:jump-host-cluster-access@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
     --role="	roles/container.admin"
   ```

2. **Firewall Rules** (from VPC creation guide) allow:
   - Health check IPs (35.191.0.0/16, 130.211.0.0/22) to reach pods on port 80 and 10254

3. **IAP Firewall Rule** allows SSH via IAP:
   ```bash
   gcloud compute firewall-rules create allow-iap-ssh \
     --network=rakops-vpc-dev \
     --action=ALLOW \
     --direction=INGRESS \
     --source-ranges=35.235.240.0/20 \
     --rules=tcp:22 \
     --target-tags=iap-ssh
   ```
4. Allow load balancer to do health checks
    ```bash
    gcloud compute firewall-rules create allow-ingress-from-loadbalancer \
        --network=rakops-vpc-dev\
        --action=ALLOW \
        --direction=INGRESS \
        --source-ranges=35.191.0.0/16,130.211.0.0/22 \
        --rules=tcp:80,tcp:10254 \
        --target-tags=gke-rakops-cluster-07c909d6-node  # network tag which is present on worker nodes 
    ```

Then add the tag to the jump host:
   ```bash
   gcloud compute instances add-tags jump-host --tags=iap-ssh --zone=asia-south1-a
   ```

### Access Jump Host via IAP

```bash
gcloud compute ssh --project=YOUR_PROJECT_ID --zone=asia-south1-a jump-host --tunnel-through-iap
```

You should now have an SSH shell inside the jump host. Next, install tools.

---

## Step 4: Install kubectl & GCP Tools on Jump Host

Once SSH'd into the jump host, install the required tools.

### Update System Packages

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

### Install kubectl

Download and install the stable kubectl binary:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### Install GCP Auth Plugin for kubectl

This plugin allows `kubectl` to authenticate to private GKE clusters using GCP credentials:

```bash
# Add Google Cloud repository
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/google-cloud.gpg
echo "deb [signed-by=/etc/apt/keyrings/google-cloud.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee /etc/apt/sources.list.d/google-cloud-sdk.list

# Install Google Cloud CLI and auth plugin
sudo apt-get update
sudo apt-get install -y google-cloud-cli google-cloud-cli-gke-gcloud-auth-plugin
```

Verify installation:

```bash
which gke-gcloud-auth-plugin
```

### Install Useful Tools (Optional)

```bash
sudo apt install -y kubectx git vim curl
```

### Get Cluster Credentials

Configure `kubectl` to connect to your GKE cluster:

```bash
gcloud container clusters get-credentials rakops-cluster \
  --region=asia-south1 \
  --project=YOUR_PROJECT_ID
```

This command:
1. Fetches the cluster's control plane endpoint
2. Retrieves TLS certificates
3. Configures `~/.kube/config` with cluster authentication

### Verify Cluster Access

```bash
kubectl get nodes
```

Success! You can now manage the cluster from the jump host.

---

## Step 5: Install & Configure Nginx Ingress Controller

The Nginx Ingress Controller acts as a reverse proxy. It reads Kubernetes Ingress objects and configures Nginx routing rules accordingly. Traffic flows: External → Google Cloud LB → Nginx Pod → Backend Services.

### 5.1 Deploy Nginx Ingress Controller via Helm

Add the Nginx Helm repository:

```bash
kubectl apply -f ingress-setup-files/ingress-controller-gcp.yaml 
```


### Verify Nginx Controller is Running

```bash
kubectl get pods -n ingress-nginx
```

Expected output:

```
NAME                                              READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-xyz-abc                  1/1     Running   0          2m
ingress-nginx-admission-create-xyz-abc            0/1     Completed 0          2m
ingress-nginx-admission-patch-xyz-abc             0/1     Completed 0          2m
```

### Verify NEG Was Created

The annotation should automatically create a Network Endpoint Group (NEG) in GCP:

```bash
gcloud compute network-endpoint-groups list --region=asia-south1
```

---

## Step 6: Configure Google Cloud Load Balancer

Now that Nginx is running, configure the Google Cloud **external load balancer** to route external traffic to the Nginx controller.

### 6.1 Create a Static Global IP Address

The load balancer needs a static IP that doesn't change:

```bash
gcloud compute addresses create rakops-loadbalancer-ip \
    --global \
    --ip-version IPV4 \
    --project=YOUR_PROJECT_ID
```

Retrieve the IP:

```bash
gcloud compute addresses describe rakops-loadbalancer-ip --global --project=YOUR_PROJECT_ID
```

**Save this IP** — you'll need it for DNS configuration later.

### 6.2 Create Health Check

The load balancer needs to verify Nginx pods are healthy. It checks the metrics endpoint on port 10254:

```bash
gcloud compute health-checks create http rakops-health-check \
    --global \
    --region=asia-south1 \
    --port=10254 \
    --request-path=/healthz \
    --proxy-header=NONE \
    --check-interval=30s \
    --timeout=5s \
    --healthy-threshold=2 \
    --unhealthy-threshold=2 \
    --project=YOUR_PROJECT_ID
```

### Parameter Breakdown

| Flag | Purpose |
|------|---------|
| `--port=10254` | Nginx metrics port (not 80 — traffic goes there directly) |
| `--request-path=/healthz` | Health check endpoint (Nginx standard) |
| `--check-interval=30s` | Check every 30 seconds |
| `--healthy-threshold=2` | Pod must pass 2 checks to be "healthy" |

### 6.3 Create Backend Service

The backend service groups the NEG (containing Nginx pods) for load balancing:

```bash
gcloud compute backend-services create rakops-backend-service \
    --global \
    --load-balancing-scheme=EXTERNAL_MANAGED \
    --protocol=HTTP \
    --health-checks=rakops-health-check \
    --timeout=30s \
    --ip-address-selection-policy=IPV4_ONLY \
    --no-enable-cdn \
    --no-enable-logging \
    --project=YOUR_PROJECT_ID
```

### 6.4 Add NEG Backends to Service

The NEG is deployed across multiple zones. Add each zone's NEG to the backend service:

```bash
# Zone asia-south1-a
gcloud compute backend-services add-backend rakops-backend-service \
    --global \
    --network-endpoint-group=ingress-nginx-http-neg \
    --network-endpoint-group-zone=asia-south1-a \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=100 \
    --capacity-scaler=1.0 \
    --project=YOUR_PROJECT_ID

# Zone asia-south1-b
gcloud compute backend-services add-backend rakops-backend-service \
    --global \
    --network-endpoint-group=ingress-nginx-http-neg \
    --network-endpoint-group-zone=asia-south1-b \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=100 \
    --capacity-scaler=1.0 \
    --project=YOUR_PROJECT_ID

# Zone asia-south1-c
gcloud compute backend-services add-backend rakops-backend-service \
    --global \
    --network-endpoint-group=ingress-nginx-http-neg \
    --network-endpoint-group-zone=asia-south1-c \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=100 \
    --capacity-scaler=1.0 \
    --project=YOUR_PROJECT_ID
```

### Parameter Breakdown

| Flag | Purpose |
|------|---------|
| `--balancing-mode=RATE` | Scale based on requests per second |
| `--max-rate-per-endpoint=100` | Max 100 RPS per pod (adjust based on needs) |
| `--capacity-scaler=1.0` | Use 100% of capacity (no throttling) |

### 6.5 Create URL Map

The URL map defines routing rules (which requests go to which backend):

```bash
gcloud compute url-maps create rakops-url-map \
    --default-service=rakops-backend-service \
    --project=YOUR_PROJECT_ID
```


### 6.6 Create HTTPS Proxy (for HTTPS)

First, upload or reference a certificate. Create a self-managed certificate:

```bash
gcloud compute ssl-certificates create rakops-certificate \
    --certificate=path/to/cert.crt \
    --private-key=path/to/key.key \
    --project=YOUR_PROJECT_ID
```

Or create a self-signed certificate for testing:

```bash
openssl req -x509 -newkey rsa:2048 -nodes -keyout key.key -out cert.crt -days 365 -subj "/CN=rakops.in"

gcloud compute ssl-certificates create rakops-certificate \
    --certificate=cert.crt \
    --private-key=key.key \
    --project=YOUR_PROJECT_ID
```

Create HTTPS proxy:

```bash
gcloud compute target-https-proxies create rakops-https-proxy \
    --url-map=rakops-url-map \
    --ssl-certificates=rakops-certificate \
    --ssl-policy=default-security-policy-for-rakops \
    --project=YOUR_PROJECT_ID
```

### 6.8 Create HTTP Proxy (for HTTP redirects)

```bash
gcloud compute target-http-proxies create rakops-http-proxy \
    --url-map=rakops-url-map \
    --project=YOUR_PROJECT_ID
```

### 6.9 Create Forwarding Rules

Create HTTP forwarding rule (redirects to HTTPS):

```bash
gcloud compute forwarding-rules create rakops-http-rule \
    --global \
    --target-http-proxy=rakops-http-proxy \
    --address=rakops-loadbalancer-ip \
    --ports=80 \
    --project=YOUR_PROJECT_ID
```

Create HTTPS forwarding rule:

```bash
gcloud compute forwarding-rules create rakops-https-rule \
    --global \
    --target-https-proxy=rakops-https-proxy \
    --address=rakops-loadbalancer-ip \
    --ports=443 \
    --project=YOUR_PROJECT_ID
```

### 6.10 Verify Load Balancer

```bash
gcloud compute forwarding-rules list --global --project=YOUR_PROJECT_ID
```

Expected output:

```
NAME                     REGION   IP_ADDRESS       IP_PROTOCOL   TARGET
rakops-http-rule         -        XX.XXX.XXX.XXX   TCP           rakops-http-proxy
rakops-https-rule        -        XX.XXX.XXX.XXX   TCP           rakops-https-proxy
```

### 6.11 Check Backend Health

```bash
gcloud compute backend-services get-health rakops-backend-service --global --project=YOUR_PROJECT_ID
```

Expected output (NEGs should be HEALTHY):

```
---
backend: https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT_ID/global/networkEndpointGroups/ingress-nginx-http-neg/zones/asia-south1-a
status:
  healthStatus:
  - healthState: HEALTHY
    instance: https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT_ID/zones/asia-south1-a/instances/gke-rakops-cluster-default-pool-xxx-yyy
    ipAddress: 10.0.1.5
    port: 80
```

If backends show `UNHEALTHY`, check:
- Firewall rules allow health check IPs (35.191.0.0/16, 130.211.0.0/22) on port 10254
- Nginx pod is actually running: `kubectl get pods -n ingress-nginx`
- Pod logs: `kubectl logs -n ingress-nginx deployment/ingress-nginx-controller`

---

## Step 7: Configure DNS

Point your domain to the load balancer IP.

### Get Load Balancer IP

```bash
gcloud compute addresses describe rakops-loadbalancer-ip --global --project=YOUR_PROJECT_ID --format='value(address)'
```

Example output: `35.232.123.45`

### Update DNS Records

In your domain registrar or DNS provider (GCP Cloud DNS, Route 53, etc.), create:

**A Record:**
- **Name:** vault.rakops.in (or your subdomain)
- **Type:** A
- **Value:** 35.232.123.45 (from above)


- **Name:** vault.rakops.in
- **Type:** A
- **Value:** 35.232.123.45 (from above)

### Verify DNS Resolution

```bash
nslookup vault.rakops.in
```

Should resolve to your load balancer IP.

### Wait for Propagation

DNS propagation can take 15-60 minutes. Test with:

```bash
curl https://vault.rakops.in
```

Initially, you'll get a certificate error (expected, since we used self-signed cert). In production, use a real certificate.

---

## Step 7: Deploy HashiCorp Vault (Demo App)

Deploy Vault as a sample stateful application to test the full stack.

### Add HashiCorp Helm Repository

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

Search available versions:

```bash
helm search repo hashicorp/vault -l
```

### Create Namespace & Install

```bash
kubectl create ns vault

helm install vault hashicorp/vault --namespace vault --version 0.34.1
```


## Create ingress object 

``` bash
kubectl apply -f vault-ingress.yaml
### Initialize Vault

Open a shell in the Vault pod:

```bash
kubectl exec -it vault-0 -n vault -- sh
```

Inside the pod, run: 

```bash
vault operator init
vault operator unseal
```

**Critical:** Save the unseal keys and root token securely. Do **NOT** commit to version control or Slack. Store in a secure password manager.

---
## Step 8: Deploy Jenkins (Demo App - optional)

### Create Namespace & Install
```bash
kubectl create ns jenkins
helm install my-jenkins jenkins/jenkins  -n jenkins

kubectl apply -f jenkins-ingress.yaml
```

---


## Troubleshooting

### Cluster Access Failed: "couldn't connect to server"

**Cause:** Jump host doesn't have `gke-gcloud-auth-plugin` installed.  
**Fix:** Run on jump host:
```bash
sudo apt-get install -y google-cloud-cli-gke-gcloud-auth-plugin
```

### Backend Health Check Failing

**Cause:** Firewall rules don't allow health check IPs.  
**Fix:** Verify firewall allows:
- Source: `35.191.0.0/16`, `130.211.0.0/22`
- Destination ports: `80`, `10254`
- Target: GKE node network tag

```bash
gcloud compute firewall-rules list --filter="network:rakops-vpc-dev"
```

### NEG Shows 0 Endpoints

**Cause:** Annotation missing on Nginx service.  
**Fix:** Verify annotation:
```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.metadata.annotations}' | jq .
```

Should contain: `"cloud.google.com/neg": "{\"exposed_ports\": {\"80\":{\"name\": \"ingress-nginx-http-neg\"}}}"`

### DNS Not Resolving

**Cause:** DNS propagation delay or wrong IP.  
**Fix:** Verify DNS points to correct LB IP:
```bash
dig vault.rakops.in
gcloud compute addresses describe rakops-loadbalancer-ip --global --format='value(address)'
```


# GCP GKE Ingress Migration: Nginx Controller → Envoy Gateway API

## Migrating to Modern Standards-Based Gateway API

---

## Overview

This guide walks through migrating from the traditional **Nginx Ingress Controller** to **Envoy Gateway**—a modern, standards-based alternative implementing the Kubernetes Gateway API.

### Why Migrate to Envoy Gateway?

| Feature | Nginx Ingress | Envoy Gateway |
|---------|---------------|---------------|
| API Standard | Legacy Ingress API | Gateway API (k8s.io/v1beta1) |
| Routing Semantics | Basic (host, path) | Advanced (HTTPRoute, matching conditions) |
| Traffic Control | Limited | Traffic splitting, weights, retries |
| Request Modification | Via annotations | Native HTTP filters |
| Observability | Basic logs | Envoy metrics, tracing, debugging |
| Extensibility | Plugins/annotations | Gateway filters, security policies |
| Industry Adoption | Legacy (deprecating) | Future standard (CNCF) |

### By the End of This Guide

- Envoy Gateway control plane running in your cluster
- Gateway data plane pods managing traffic routing
- Network Endpoint Groups (NEG) for dynamic pod discovery
- HTTPRoutes configured for applications (Vault, Jenkins)
- Load balancer pointing to Envoy backend service
- DNS resolving to applications via Envoy Gateway

**Migration Path:** Keep Nginx running during migration → Test Envoy routes → Switch load balancer → Remove Nginx

---

## Prerequisites

Before starting, ensure you have completed:

- ✅ Nginx Ingress Controller running in `ingress-nginx` namespace
- ✅ Google Cloud LB pointing to Nginx NEG
- ✅ Applications deployed with Nginx Ingress objects
- ✅ `kubectl` and `gcloud` CLI access on jump host
- ✅ Helm 3+ installed

**Important:** This guide does **NOT** remove Nginx. You can run both simultaneously during testing.

---

## Step 1: Install Envoy Gateway Control Plane

Envoy Gateway consists of two parts:
1. **Control Plane** — Watches Gateway/HTTPRoute resources, configures Envoy proxies
2. **Data Plane** — Actual Envoy proxy pods handling traffic routing

### 1.1 Install Envoy Gateway Controller

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.9.1   -n envoy-gateway-system   --create-namespace
```

**Note:** Replace version `1.9.1` with the latest stable version from the [Envoy Gateway releases](https://github.com/envoyproxy/gateway/releases).

### 1.2 Verify Control Plane

```bash
kubectl get pods -n envoy-gateway-system
```

Expected output:

```
NAME                                 READY   STATUS    RESTARTS   AGE
envoy-gateway-5f8b9c7d9d-abc2x       1/1     Running   0          2m
```

The control plane pods are now running and watching for Gateway/HTTPRoute resources.

---

## Step 2: Configure EnvoyProxy Custom Resource

The **EnvoyProxy** resource defines how Envoy proxies behave—pod replicas, resource limits, network configuration, etc. Importantly, it adds the annotation to create a **Network Endpoint Group (NEG)** so Google Cloud LB can discover Envoy pods.

### 2.1 Create EnvoyProxy Manifest

Create a file named `envoy-proxy-gcp.yaml`:

  
  # Pod annotations for GCP integration
  podAnnotations:
    cloud.google.com/neg: '{"exposed_ports": {"10080":{"name": "gateway-envoy-http-neg"}}}'
  
```

### 2.2 Apply EnvoyProxy Configuration

```bash
kubectl apply -f envoy-proxy-gcp.yaml
```

### Key Configuration Points

| Parameter | Purpose |
|-----------|---------|
| `cloud.google.com/neg` annotation | **Critical** — Creates NEG for load balancer discovery |




---

## Step 3: Create GatewayClass & Gateway Resources

**GatewayClass** tells Kubernetes which controller manages Gateway objects (similar to IngressClass for Ingress). **Gateway** provisions actual Envoy proxy pods.

### 3.1 Create GatewayClass Manifest



Apply it:

```bash
kubectl apply -f gateway-class.yaml
```

### 3.2 Create Gateway Manifest (Data Plane)


Apply it:

```bash
kubectl apply -f gateway.yaml
```

### 3.3 Verify Data Plane Pods Created

The Gateway controller automatically creates Envoy proxy pods:

```bash
kubectl get pods -n envoy-gateway-system
```

Expected output:

```
NAME                                                            READY   STATUS    RESTARTS   AGE
envoy-envoy-gateway-system-main-gateway-c3508b54-86b95d79d...   2/2     Running   0          30s
envoy-gateway-6fbfccc98d-q86rd                                  1/1     Running   0          5m
```

The pod name format: `envoy-{namespace}-{gateway-name}-{hash}`.

### 3.4 Verify Data Plane Service

```bash
kubectl get svc -n envoy-gateway-system
```

Expected output:

```
NAME                                         TYPE        CLUSTER-IP       PORT(S)      AGE
envoy-envoy-gateway-system-main-gateway      ClusterIP   10.100.50.XX     10080/TCP    15s
envoy-gateway                                ClusterIP   10.100.50.YY     9001/TCP     5m
```

The data plane service is **ClusterIP** (internal) because Google Cloud LB discovers pods via NEG, not via service.

### 3.5 Verify NEG Created in GCP

The annotation in EnvoyProxy should have automatically created Network Endpoint Groups. Verify in GCP Console:

```bash
gcloud compute network-endpoint-groups list --region=asia-south1
```

Expected output:

```
NAME                        REGION         NETWORK_ENDPOINT_TYPE   SIZE
gateway-envoy-http-neg      asia-south1    GCE_VM_IP_PORT          1
ingress-nginx-http-neg      asia-south1    GCE_VM_IP_PORT          1
```

---

## Step 4: Create HTTPRoutes for Applications

**HTTPRoutes** replace the traditional Ingress objects. They provide fine-grained traffic routing with better semantics (matching rules, weights, request modification).

### 4.1 Create Vault HTTPRoute

Apply it:

```bash
kubectl apply -f vault-httproute.yaml
```

### 4.2 Create Jenkins HTTPRoute


Apply it:

```bash
kubectl apply -f jenkins-httproute.yaml
```

### 4.3 Verify HTTPRoutes are Bound

```bash
kubectl get httproute -A
```

Expected output:

```
NAMESPACE   NAME             HOSTNAMES              PARENTS   AGE
vault       vault-route      [vault.rakops.in]      1         10s
jenkins     jenkins-route    [jenkins.rakops.in]    1         5s
```

Check route status:

```bash
kubectl describe httproute vault-route -n vault
```

Should show `Accepted: True` in the status.

---

## Step 5: Update Google Cloud Load Balancer Configuration

Now reconfigure the load balancer to point to the **Envoy backend service** instead of Nginx. You'll keep the same load balancer infrastructure (forwarding rules, IP, SSL) but swap the backend.

### 5.1 Update Firewall Rules

Allow traffic from Google Cloud LB health checks to Envoy pods on ports 10080 (traffic) and 19003 (health):

```bash
gcloud compute firewall-rules create allow-envoy-ingress-from-lb \
    --network=rakops-vpc-dev \
    --action=ALLOW \
    --direction=INGRESS \
    --source-ranges=35.191.0.0/16,130.211.0.0/22 \
    --rules=tcp:10080,tcp:19003 \
    --target-tags=gke-rakops-cluster-fc8dabb5-node \
    --description="Allow GCP LB and health checks to Envoy on 10080 and 19003" \
    --project=YOUR_PROJECT_ID
```

**Note:** Replace `gke-rakops-cluster-fc8dabb5-node` with your actual GKE node network tag. Find it with:

```bash
gcloud compute instances list --filter="zone:asia-south1-*" --format="value(tags.items[0])"
```

### 5.2 Create Envoy Health Check

```bash
gcloud compute health-checks create http envoy-proxy-health-check \
    --global \
    --port=19003 \
    --request-path=/ready \
    --check-interval=30s \
    --timeout=5s \
    --healthy-threshold=2 \
    --unhealthy-threshold=2 \
    --description="Health check for Envoy proxy on admin port 19003" \
    --project=YOUR_PROJECT_ID
```

**Why port 19003?** Envoy admin interface runs on :19003 (configurable but standard). The `/ready` endpoint returns 200 when Envoy is ready.

### 5.3 Create New Backend Service for Envoy

```bash
gcloud compute backend-services create rakops-backend-envoy-service \
    --global \
    --load-balancing-scheme=EXTERNAL_MANAGED \
    --protocol=HTTP \
    --health-checks=envoy-proxy-health-check \
    --timeout=30s \
    --ip-address-selection-policy=IPV4_ONLY \
    --no-enable-cdn \
    --no-enable-logging \
    --project=YOUR_PROJECT_ID
```

### 5.4 Add NEG Backends Across All Zones

Add the Envoy NEG to the backend service for each zone:

```bash
# Zone asia-south1-a
gcloud compute backend-services add-backend rakops-backend-envoy-service \
    --global \
    --network-endpoint-group=gateway-envoy-http-neg \
    --network-endpoint-group-zone=asia-south1-a \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=100 \
    --capacity-scaler=1.0 \
    --project=YOUR_PROJECT_ID

# Zone asia-south1-b
gcloud compute backend-services add-backend rakops-backend-envoy-service \
    --global \
    --network-endpoint-group=gateway-envoy-http-neg \
    --network-endpoint-group-zone=asia-south1-b \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=100 \
    --capacity-scaler=1.0 \
    --project=YOUR_PROJECT_ID

# Zone asia-south1-c
gcloud compute backend-services add-backend rakops-backend-envoy-service \
    --global \
    --network-endpoint-group=gateway-envoy-http-neg \
    --network-endpoint-group-zone=asia-south1-c \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=100 \
    --capacity-scaler=1.0 \
    --project=YOUR_PROJECT_ID
```

### 5.5 Update URL Map to Use Envoy Backend

If you created a URL map during Nginx setup, update it to point to the Envoy backend:

```bash
# Update the default service
gcloud compute url-maps update rakops-url-map \
    --default-service=rakops-backend-envoy-service \
    --project=YOUR_PROJECT_ID
```

### 5.6 Verify Envoy Backend Health

```bash
gcloud compute backend-services get-health rakops-backend-envoy-service --global --project=YOUR_PROJECT_ID
```

Expected output (NEGs should be HEALTHY):

```
---
backend: https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT_ID/global/networkEndpointGroups/gateway-envoy-http-neg/zones/asia-south1-a
status:
  healthStatus:
  - healthState: HEALTHY
    instance: https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT_ID/zones/asia-south1-a/instances/gke-rakops-cluster-default-pool-xxx-yyy
    ipAddress: 10.0.1.5
    port: 10080
```

If backends show `UNHEALTHY`:
- Verify Envoy pod is running: `kubectl get pods -n envoy-gateway-system`
- Check pod logs: `kubectl logs -n envoy-gateway-system -l app.kubernetes.io/name=envoy`
- Verify firewall rule is applied to correct node tags

---

## Step 6: Test Envoy Routes

Once the backend is healthy, test traffic routing.

### 6.1 Test Vault

```bash
curl -I https://vault.rakops.in
```

Expected output:

```
HTTP/2 200
```

If you get 502 Bad Gateway:
- Wait for backend to become HEALTHY (takes 1-2 min)
- Check HTTPRoute is `Accepted`: `kubectl describe httproute vault-route -n vault`
- Check service exists: `kubectl get svc -n vault`

### 6.2 Test Jenkins

```bash
curl -I https://jenkins.rakops.in
```

Expected output:

```
HTTP/2 200
```

### 6.3 Verify Traffic is Routing Through Envoy

Check Envoy pod access logs:

```bash
kubectl logs -n envoy-gateway-system -l app.kubernetes.io/name=envoy --tail=20
```

Should show HTTP requests from the load balancer.

---

## Step 7: Migrate Traffic to Envoy

Once Envoy is working and routes are verified, switch the load balancer from Nginx to Envoy. **You can keep Nginx running as a fallback.**

### 7.1 Switch Load Balancer (Already Done in Step 5.5)

If you updated the URL map, traffic is already flowing through Envoy. The old Nginx backend service (`rakops-backend-service`) is no longer receiving traffic.

### 7.2 Optional: Remove Nginx from Load Balancer

If you want to confirm Nginx is not being used:

```bash
gcloud compute backend-services get-health rakops-backend-service --global --project=YOUR_PROJECT_ID
```

If backends show `UNHEALTHY` or the backend service is not referenced by any URL map, Nginx is not receiving traffic.

### 7.3 Keep Nginx Running (Recommended)

For 1-2 weeks, keep Nginx running as a fallback:
- Quickly rollback if Envoy has issues
- Test both stacks simultaneously
- Monitor Envoy metrics and logs

After confidence period, you can delete Nginx Ingress Controller:

```bash
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete ns ingress-nginx
```

---

## Verification Checklist

Before declaring migration complete, verify:

- [ ] Envoy control plane pods running: `kubectl get pods -n envoy-gateway-system`
- [ ] Envoy data plane pods running: `kubectl get pods -n envoy-gateway-system -l app.kubernetes.io/name=envoy`
- [ ] Gateway resource created: `kubectl get gateway -A`
- [ ] HTTPRoutes created and accepted: `kubectl get httproute -A`
- [ ] NEG exists: `gcloud compute network-endpoint-groups list --region=asia-south1`
- [ ] Backend service healthy: `gcloud compute backend-services get-health rakops-backend-envoy-service --global`
- [ ] Vault accessible: `curl -I https://vault.rakops.in` → HTTP 200
- [ ] Jenkins accessible: `curl -I https://jenkins.rakops.in` → HTTP 200
- [ ] Load balancer forwarding traffic to Envoy: Check LB logs in GCP Console

---

## References

- [Envoy Gateway Official Documentation](https://gateway.envoyproxy.io/)
- [Kubernetes Gateway API](https://gateway.api.k8s.io/)
- [GCP Network Endpoint Groups](https://cloud.google.com/kubernetes-engine/docs/how-to/network-endpoint-groups)
- [Envoy Proxy Documentation](https://www.envoyproxy.io/docs/envoy/latest)
- [HTTPRoute API Reference](https://gateway.networking.k8s.io/v1beta1/api-types/)
- [GKE Official Documentation](https://cloud.google.com/kubernetes-engine/docs)
- [VPC Creation Guide](https://github.com/Rakshitsen/kubernetes-ha-gcp/blob/main/guide/01-create-vpc.md)
- [Nginx Ingress Controller Helm Chart](https://kubernetes.github.io/ingress-nginx/)
- [GCP Load Balancer Overview](https://cloud.google.com/load-balancing/docs/https)
- [Network Endpoint Groups (NEG)](https://cloud.google.com/kubernetes-engine/docs/how-to/network-endpoint-groups)

---
