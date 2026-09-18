# AWS EKS Setup & Kubernetes Ingress to Gateway API Migration


---

## Overview

This guide walks through setting up a complete Kubernetes infrastructure on AWS EKS with ingress controllers, load balancers, storage, and a sample Vault application. By the end, you will have:

- A working EKS cluster with auto-scaling nodes
- Nginx Ingress Controller routing external traffic
- AWS Load Balancer Controller managing ALBs
- EBS storage provisioning for persistent data
- HashiCorp Vault running as a demo application

---

## Prerequisites

Before starting, ensure you have:

- AWS account with programmatic access (credentials configured)
- eksctl CLI installed
- kubectl installed
- Helm 3+ installed
- SSH key pair generated (e.g., `~/.ssh/my-eks-key.pub`)
- Domain registered (for DNS records)
- AWS CLI configured with credentials
- Basic understanding of Kubernetes and AWS services

---

## Step 1: Create EKS Cluster

Create a test EKS cluster in the ap-south-1 region with Ubuntu 24.04 nodes.

### Command

```bash
eksctl create cluster \
  --name testing-cluster \
  --region ap-south-1 \
  --version 1.36 \
  --node-type t3.small \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 1 \
  --node-volume-size 25 \
  --node-ami-family Ubuntu2404 \
  --spot \
  --ssh-access \
  --ssh-public-key ~/.ssh/my-eks-key.pub
```

### Key Parameters Explained

- `--spot`: Uses spot instances for cost savings (up to 70% cheaper)
- `--node-ami-family Ubuntu2404`: Uses Ubuntu 24.04 as the base OS
- `--node-volume-size 25`: 25 GB root volume per node
- `--ssh-access`: Enables SSH access to nodes for debugging
- `--nodes-min/max 1`: Auto-scaling configuration (start with 1, scale between 1-1)

**Timeline:** This command takes 10-15 minutes. The cluster is ready when all nodes are in `Running` state.

---

## Step 2: Install & Configure Nginx Ingress Controller

The Nginx Ingress Controller acts as a reverse proxy. It reads Kubernetes Ingress objects and configures Nginx routing rules accordingly. Traffic flows: External → ALB → Nginx Pod → Backend Services.

### 2.1 Deploy Nginx Ingress Controller

Create an `ingress-controller-aws.yaml` manifest and apply it:

```bash
kubectl apply -f ingress-controller-aws.yaml
```

Verify the controller pod is running:

```bash
kubectl get pods -n ingress-nginx
```

Expected output:

```
NAME                                       READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-bd475d7b4-6jq2m   1/1     Running   0          69s
```

### 2.2 Create Target Group for Nginx Health Checks

The target group connects the ALB to the Nginx controller pod. Health checks use port 10254 (Nginx status/metrics port).

```bash
aws elbv2 create-target-group \
  --name ingress-nginx-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-0f97aa6bb82701a4e \
  --target-type ip \
  --health-check-protocol HTTP \
  --health-check-port 10254 \
  --health-check-path /healthz \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 \
  --matcher HttpCode=200
```

**Important:** Replace `vpc-0f97aa6bb82701a4e` with your VPC ID (find it in the EKS cluster details).

### 2.3 Bind Target Group to Nginx Service

Create `ingress-nginx-tgb.yaml` to bind the target group to the Nginx service. This ensures the target group routes traffic directly to the Nginx controller pod IPs.

```bash
# Update the VPC ID and target group ARN in the manifest first
kubectl apply -f ingress-nginx-tgb.yaml
```

This binding ensures stable routing even when pod IPs change.

---

## Step 3: Install AWS Load Balancer Controller

The AWS Load Balancer Controller automates ALB/NLB provisioning from Kubernetes Ingress objects. Instead of manually creating ALBs, the controller watches for Ingress objects and creates infrastructure automatically.

### 3.1 Configure IAM Permissions

Download the IAM policy:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
```

Create the IAM policy:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### 3.2 Setup OIDC Provider & Service Account

Enable OIDC federation (securely link Kubernetes service accounts to IAM roles):

```bash
eksctl utils associate-iam-oidc-provider \
  --region=ap-south-1 \
  --cluster=testing-cluster \
  --approve
```

Create the service account:

```bash
eksctl create iamserviceaccount \
  --cluster=testing-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

**Replace:** `ACCOUNT_ID` with your AWS account ID (find in AWS Console under Account).

### 3.3 Install via Helm

Add the EKS Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Install the controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=testing-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.14.0
```

Verify installation:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## Step 4: Configure Security Groups & ALB

### 4.1 Create Security Group for ALB

Create a security group with the following rules:

**Inbound:**
- Port 80 from `0.0.0.0/0` (HTTP from anywhere)
- Port 443 from `0.0.0.0/0` (HTTPS from anywhere)

**Outbound:**
- All traffic (default)

### 4.2 Update Worker Node Security Group

The EKS cluster automatically creates a security group: `eks-cluster-sg-testing-cluster-xxx`. Add:

**Inbound:**
- Port 10254 from ALB security group (Nginx health checks)
- Port 80 from ALB security group (HTTP traffic)

This allows the ALB to reach the Nginx controller pod.

### 4.3 Create Internet-Facing ALB

Create an Application Load Balancer:

- **Name:** testing-loadbalancer
- **Scheme:** Internet-facing
- **VPC:** Select your EKS VPC
- **Subnets:** Public subnets (EKS uses multi-AZ by default)
- **Security Group:** ALB security group created above

### 4.4 Configure Listeners & Rules

**HTTP Listener (Port 80):**
- Default action: Redirect to HTTPS
- Status code: HTTP 301

**HTTPS Listener (Port 443):**
- Certificate: ACM certificate for your domain
- Security policy: `ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09`
- Default target group: `ingress-nginx-tg`

---

## Step 5: Configure DNS

Point your domain to the ALB:

- **Record type:** CNAME
- **Name:** vault.rakops.in (or your subdomain)
- **Value:** testing-loadbalancer-1667964758.ap-south-1.elb.amazonaws.com
- **SSL mode:** Full (Strict) if using HTTPS endpoint

**Important:** If setting up SSL redirection at your DNS provider, ensure Full (Strict) mode to avoid redirect loops.

---

## Step 6: Install EBS CSI Driver (Optional Storage)

The EBS CSI driver enables dynamic persistent volume provisioning for stateful applications. Only install if your workloads need persistent storage.

### 6.1 Download & Create IAM Policy

```bash
curl -o ebs-csi-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json
```

```bash
aws iam create-policy \
  --policy-name AmazonEBSCSIDriverPolicy \
  --policy-document file://ebs-csi-policy.json
```

### 6.2 Get OIDC Provider ID

```bash
aws eks describe-cluster --name testing-cluster --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f5
```

Save the returned ID (e.g., `6FF7AD29615C0E9B23C9C11790E7733F`) for the next step.

### 6.3 Create IAM Role for EBS CSI

```bash
aws iam create-role \
  --role-name AmazonEBSCSIDriverRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/OIDC_ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/OIDC_ID:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }]
  }' \
  --region ap-south-1
```

**Replace:** `ACCOUNT_ID` and `OIDC_ID` from steps above.

Attach the policy:

```bash
aws iam attach-role-policy \
  --role-name AmazonEBSCSIDriverRole \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/AmazonEBSCSIDriverPolicy \
  --region ap-south-1
```

### 6.4 Install EBS CSI Driver

Add the Helm repository:

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
```

Install the driver:

```bash
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::ACCOUNT_ID:role/AmazonEBSCSIDriverRole"
```

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
## Verification Checklist

Before considering the setup complete, verify:

- [ ] `kubectl get nodes` → All nodes in `Ready` state
- [ ] `kubectl get pods -n ingress-nginx` → Nginx controller in `Running` state
- [ ] `kubectl get deployment -n kube-system aws-load-balancer-controller` → Replicas match desired count
- [ ] AWS Console → ALB is `Active` with target group `Healthy`
- [ ] `curl -k https://vault.rakops.in` → Vault UI responds (or redirects correctly)
- [ ] `kubectl get pods -n vault` → Vault pod in `Running` state

---


# AWS EKS Ingress Migration: Nginx Controller → Envoy Gateway API

## Overview

This guide walks through migrating from Nginx Ingress Controller to Envoy Gateway—a modern, standards-based alternative implementing the Kubernetes Gateway API. Envoy Gateway offers:

- **Gateway API compliance:** Uses standardized Gateway/HTTPRoute resources instead of legacy Ingress objects
- **Fine-grained traffic control:** HTTPRoutes provide better routing semantics than Ingress
- **Advanced features:** Traffic splitting, request/response modification, circuit breaking
- **Better observability:** Native integration with Envoy Proxy's metrics and tracing

By the end of this section, you will have:

- Envoy Gateway control plane running in the cluster
- Gateway data plane pods managing traffic routing
- Target group binding to ALB (via TargetGroupBinding)
- HTTPRoutes configured for Vault and Jenkins applications
- Updated security groups allowing ALB → Envoy traffic

---

## Prerequisites

Before starting, ensure you have:

- Existing EKS cluster (from Steps 1–6 of the main guide)
- AWS CLI configured
- kubectl and Helm installed
- Vault and Jenkins applications still running (they will route through Envoy instead of Nginx)
- VPC ID and ALB security group ID available
- OIDC provider already configured (from Step 3.2)

---

## Step 1: Install Envoy Gateway Control Plane

Envoy Gateway is deployed as a controller that watches Gateway and HTTPRoute resources and provisions Envoy proxies.

### 1.1 Add Helm Repository & Install

Add the Envoy Gateway Helm repository:

```bash
helm repo add envoy https://gateway.envoyproxy.io
helm repo update envoy
```

Install Envoy Gateway:

```bash
helm install eg envoy/gateway --version v1.9.1 \
  -n envoy-gateway-system \
  --create-namespace
```

### 1.2 Verify Installation

Confirm the control plane pod is running:

```bash
kubectl get pods -n envoy-gateway-system
```

Expected output:

```
NAME                            READY   STATUS    RESTARTS   AGE
envoy-gateway-6fbfccc98d-q86rd   1/1     Running   0          45s
```

Also verify the service exists:

```bash
kubectl get svc -n envoy-gateway-system
```

Output should show `envoy-gateway` service (type `ClusterIP`).

---

## Step 2: Configure EnvoyProxy & GatewayClass

The EnvoyProxy custom resource defines how Envoy proxies behave. The GatewayClass tells Kubernetes which controller manages Gateway objects.

### 2.1 Create EnvoyProxy Configuration

Apply the configuration:

```bash
kubectl apply -f envoy-proxy.yaml
```

Verify the EnvoyProxy resource:

```bash
kubectl get EnvoyProxy -n envoy-gateway-system
```

### 2.2 Create GatewayClass

Create `gateway-class.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

Apply the GatewayClass:

```bash
kubectl apply -f gateway-class.yaml
```

Verify it's accepted:

```bash
kubectl get gatewayclass
```

Expected output:

```
NAME            CONTROLLER                                      ACCEPTED   AGE
envoy-gateway   gateway.envoyproxy.io/gatewayclass-controller   True       12s
```

**Key point:** `ACCEPTED: True` means the Envoy Gateway controller is managing this class.

---

## Step 3: Create Gateway (Data Plane)

The Gateway resource provisions actual Envoy proxy pods that handle traffic routing. This is the data plane.

### 3.1 Create Gateway Manifest


Apply the Gateway:

```bash
kubectl apply -f gateway.yaml
```

### 3.2 Verify Data Plane Pods

Check that Envoy data plane pods are created:

```bash
kubectl get pods -n envoy-gateway-system
```

Expected output:

```
NAME                                                            READY   STATUS    RESTARTS   AGE
envoy-envoy-gateway-system-main-gateway-c3508b54-86b95d79d...   2/2     Running   0          30s
envoy-gateway-6fbfccc98d-q86rd                                  1/1     Running   0          5m
```

The pod name format is: `envoy-{namespace}-{gateway-name}-{hash}`.

Verify the data plane service is created:

```bash
kubectl get svc -n envoy-gateway-system
```

Expected output includes:

```
NAME                                         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
envoy-envoy-gateway-system-main-gateway      ClusterIP   10.100.50.XX   <none>        10080/TCP 15s
```

The service is type `ClusterIP` (not LoadBalancer) because AWS ALB will route to it via the target group binding.

---

## Step 4: Create AWS Target Group for Envoy Data Plane

The target group connects the ALB to Envoy proxy pod IPs. Health checks use Envoy's admin port (19003).

### 4.1 Create Target Group

```bash
aws elbv2 create-target-group \
  --name envoy-proxy-tg \
  --protocol HTTP \
  --port 10080 \
  --vpc-id vpc-05fde4da728047e21 \
  --target-type ip \
  --health-check-protocol HTTP \
  --health-check-port 19003 \
  --health-check-path /ready \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 \
  --matcher HttpCode=200
```

**Replace:** `vpc-05fde4da728047e21` with your EKS VPC ID.

**Health check details:**
- Port 19003: Envoy admin interface
- Path `/ready`: Readiness probe endpoint
- Matcher `HttpCode=200`: Target is healthy when it responds with 200

**Save the output:** Note the `TargetGroupArn` for the next step.

Example: `arn:aws:elasticloadbalancing:ap-south-1:ACCOUNT_ID:targetgroup/envoy-proxy-tg/50dc6c3c1edc0569`

### 4.2 Bind Target Group to Envoy Service

**Replace:**
- `targetGroupARN`: Use the ARN from step 4.1
- `serviceRef.name`: Must match the Envoy data plane service name (from step 3.2)
- `vpc id`

Apply the binding:

```bash
kubectl apply -f gateway-envoy-tgb.yaml
```

Verify the binding:

```bash
kubectl get TargetGroupBinding -n envoy-gateway-system
```

Expected output:

```
NAME                 SERVICE-NAME                            TARGET-GROUP-ARN                                                                      AGE
envoy-gateway-tgb    envoy-envoy-gateway-system-main-gateway  arn:aws:elasticloadbalancing:ap-south-1:ACCOUNT_ID:targetgroup/envoy-proxy-tg/...   10s
```

Check that targets are registered and healthy in AWS:

```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:ap-south-1:ACCOUNT_ID:targetgroup/envoy-proxy-tg/50dc6c3c1edc0569
```

---

## Step 5: Create HTTPRoutes for Applications

HTTPRoutes define how traffic is routed to backend services. Replace Ingress objects with HTTPRoute.

### 5.1 Create Vault HTTPRoute

Apply the route:

```bash
kubectl apply -f vault-route.yaml
```

### 5.2 Create Jenkins HTTPRoute

Apply the route:

```bash
kubectl apply -f jenkins-route.yaml
```

Verify routes are bound:

```bash
kubectl get httproute -A
```

Expected output:

```
NAMESPACE   NAME             HOSTNAMES              AGE
vault       vault-route      [vault.rakops.in]      20s
jenkins     jenkins-route    [jenkins.rakops.in]    15s
```

---

## Step 6: Update Security Groups

Traffic flow: ALB → Worker Node Security Group → Envoy Pods.

Update the worker node security group (`eks-cluster-sg-testing-cluster-xxx`) with two new inbound rules:

### 6.1 Rule 1: HTTP Traffic from ALB

- **Type:** Custom TCP
- **Protocol:** TCP
- **Port Range:** 10080
- **Source:** ALB security group (e.g., `sg-0ef7e767f1db17925`)
- **Description:** Allow ALB to reach Envoy on port 10080

Example rule ID: `sgr-00dcac805bfbb10e3`

### 6.2 Rule 2: Health Check from ALB

- **Type:** Custom TCP
- **Protocol:** TCP
- **Port Range:** 19003
- **Source:** ALB security group (e.g., `sg-0ef7e767f1db17925`)
- **Description:** Allow ALB health checks on Envoy admin port

Example rule ID: `sgr-0f95598dfc63723b7`

```bash
# Add Rule 1
aws ec2 authorize-security-group-ingress \
  --group-id sg-WORKER_NODE_SG_ID \
  --protocol tcp \
  --port 10080 \
  --source-security-group-id sg-0ef7e767f1db17925 \
  --region ap-south-1

# Add Rule 2
aws ec2 authorize-security-group-ingress \
  --group-id sg-WORKER_NODE_SG_ID \
  --protocol tcp \
  --port 19003 \
  --source-security-group-id sg-0ef7e767f1db17925 \
  --region ap-south-1
```

**Replace:** `sg-WORKER_NODE_SG_ID` with your worker node security group ID.

---

## Step 7: Update ALB Listener to Point to Envoy Target Group

Update the ALB's HTTPS listener (port 443) to route to the Envoy target group instead of the Nginx target group.

### 7.1 In AWS Console

1. Navigate to **EC2 → Load Balancers**
2. Select your ALB (e.g., `testing-loadbalancer`)
3. Click **Listeners** tab
4. Edit the **HTTPS:443** listener
5. Change the default target group from `ingress-nginx-tg` to `envoy-proxy-tg`
6. Save changes

### 7.2 Via AWS CLI

```bash
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:ap-south-1:ACCOUNT_ID:listener/app/testing-loadbalancer/50dc6c3c1edc0569/5c345a7a3e2d4b1a \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:ap-south-1:ACCOUNT_ID:targetgroup/envoy-proxy-tg/50dc6c3c1edc0569
```

**Replace:**
- `listener-arn`: Your ALB's HTTPS listener ARN
- `TargetGroupArn`: Your Envoy target group ARN

---

## Verification Checklist

Verify the migration is complete:

- [ ] `kubectl get pods -n envoy-gateway-system` → Both control plane and data plane pods in `Running`
- [ ] `kubectl get gatewayclass` → `envoy-gateway` shows `ACCEPTED: True`
- [ ] `kubectl get httproute -A` → All routes show as bound/valid
- [ ] AWS Console → Envoy target group shows targets as `Healthy`
- [ ] `curl -k https://vault.rakops.in` → Vault UI responds (now routed through Envoy)
- [ ] `curl -k https://jenkins.rakops.in` → Jenkins UI responds (now routed through Envoy)
- [ ] `kubectl logs -n envoy-gateway-system -l app=envoy` → No error logs during routing

---

## Optional: Decommission Nginx Ingress (if confident)

Once Envoy Gateway is stable, you can remove Nginx:

```bash
helm uninstall aws-load-balancer-controller -n kube-system
kubectl delete -f ingress-controller-aws.yaml
kubectl delete targetgroupbinding ingress-nginx-tgb -n ingress-nginx
aws elbv2 delete-target-group --target-group-arn <nginx-tg-arn>
```

**Caution:** Verify all applications are routing correctly through Envoy before deleting Nginx infrastructure.