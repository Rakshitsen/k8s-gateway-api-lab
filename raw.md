gcp gke Setup & Kubernetes Ingress to Gateway API Migration

for creation of vpc,subnet , router and nat using this guide
[vpc creation](https://github.com/Rakshitsen/kubernetes-ha-gcp/blob/main/guide/01-create-vpc.md)

Step 1: Create gke Cluster
for lab we are using this 
gcloud container clusters create rakops-cluster \
  --region=asia-south1 \
  --num-nodes=1 \
  --spot \
  --machine-type=e2-medium \
  --disk-type=pd-standard \
  --disk-size=25 \
  --network=rakops-vpc-dev \
  --no-enable-cloud-logging \
  --no-enable-cloud-monitoring \
  --subnetwork=mumbai-subnet \
  --enable-ip-alias \
  --enable-private-nodes \
  --enable-private-endpoint


or if you have custom create sunet for pod and services

gcloud container clusters create rakops-cluster 
--region=asia-south1 
--num-nodes=1 
--spot 
--machine-type=e2-medium 
--disk-type=pd-standard 
--disk-size=25 
--network=k8s-custom-vpc 
--no-enable-cloud-logging 
--no-enable-cloud-monitoring 
--subnetwork k8s-subnet-asia-south1 
--cluster-secondary-range-name=pods 
--services-secondary-range-name=services 
--enable-ip-alias 
--enable-private-nodes 
--enable-private-endpoint 



to access cluster create a jump host in same vpc 



gcloud compute instances create jump-host \
  --zone=asia-south1-a \
  --machine-type=e2-small \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --image-family=ubuntu-2404-lts-amd64 \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=10GB \
  --boot-disk-type=pd-standard \
  --network=rakops-dev-vpc \
  --subnet=mumbai-subnet \
  --no-address \
  --service-account=jump-host-cluster-access@rakops-lab-509318.iam.gserviceaccount.com \
  --scopes=cloud-platform


firewall rule for this vm is already present in vpc creation docs

gcloud compute ssh --project=rakops-lab-509318 --zone=asia-south1-a jump-host --tunnel-through-iap

sudo apt-get update
sudo   apt-get install google-cloud-cli-gke-gcloud-auth-plugin
sudo apt install kubectx

then 

install kubectl 
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install prerequisite packages
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates gnupg curl

# Import the Google Cloud public key
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/google-cloud.gpg

# Add the repository to your APT sources list
echo "deb [signed-by=/etc/apt/keyrings/google-cloud.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee /etc/apt/sources.list.d/google-cloud-sdk.list

sudo apt-get update
sudo apt-get install -y google-cloud-cli-gke-gcloud-auth-plugin

which gke-gcloud-auth-plugin


gcloud container clusters get-credentials <CLUSTER_NAME> --region <REGION> --project <PROJECT_ID>
kubectl get nodes


# Install & Configure Nginx Ingress Controller


The Nginx Ingress Controller acts as a reverse proxy. It reads Kubernetes Ingress objects and configures Nginx routing rules accordingly. Traffic flows: External → ALB → Nginx Pod → Backend Services.

2.1 Deploy Nginx Ingress Controller
Create an ingress-controller-gcp.yaml manifest and apply it:

kubectl apply -f ingress-controller-gcp.yaml

Verify the controller pod is running:

kubectl get pods -n ingress-nginx

Expected output:

NAME                                       READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-bd475d7b4-6jq2m   1/1     Running   0          69s



create Load balancer

gcloud compute addresses create rakops-loadbalancer-ip \
    --global \
    --ip-version IPV4


create your certificate self managed and then create certificate mapping 


create  global backend service which backend is neg which is being created when we create nginx ingress in which service we create these 
cloud.google.com/neg: '{"exposed_ports": {"80":{"name": "ingress-nginx-http-neg"}}}' this annotation on the service 

gcloud compute health-checks create http rakops-health-check \
    --region asia-south1 \
    --port 10254 \
    --request-path /healthz \
    --proxy-header NONE \
    --check-interval 30s \
    --timeout 5s \
    --healthy-threshold 2 \
    --unhealthy-threshold 2 \
    --no-enable-logging



gcloud compute backend-services create rakops-backend-service \
    --global \
    --load-balancing-scheme EXTERNAL_MANAGED \
    --protocol HTTP \
    --health-checks rakops-health-check \
    --timeout 30s \
    --ip-address-selection-policy IPV4_ONLY \
    --no-enable-cdn \
    --no-enable-logging \
    --security-policy default-security-policy-for-rakops-backend-service



# Zone asia-south1-a
gcloud compute backend-services add-backend rakops-backend-service \
    --global \
    --network-endpoint-group ingress-nginx-http-neg \
    --network-endpoint-group-zone asia-south1-a \
    --balancing-mode RATE \
    --max-rate-per-endpoint 100 \
    --capacity-scaler 1.0

# Zone asia-south1-b
gcloud compute backend-services add-backend rakops-backend-service \
    --global \
    --network-endpoint-group ingress-nginx-http-neg \
    --network-endpoint-group-zone asia-south1-b \
    --balancing-mode RATE \
    --max-rate-per-endpoint 100 \
    --capacity-scaler 1.0

# Zone asia-south1-c
gcloud compute backend-services add-backend rakops-backend-service \
    --global \
    --network-endpoint-group ingress-nginx-http-neg \
    --network-endpoint-group-zone asia-south1-c \
    --balancing-mode RATE \
    --max-rate-per-endpoint 100 \
    --capacity-scaler 1.0

allow load balancer to do health checks

gcloud compute firewall-rules create allow-ingress-from-loadbalancer \
    --network=rakops-vpc-dev\
    --action=ALLOW \
    --direction=INGRESS \
    --source-ranges=35.191.0.0/16,130.211.0.0/22 \
    --rules=tcp:80,tcp:10254 \
    --target-tags=gke-rakops-cluster-07c909d6-node  # network tag which is present in worker nodes 





Step 7: Deploy HashiCorp Vault (Demo App)
Deploy Vault as a sample stateful application to test the full stack.

Add HashiCorp Helm Repository
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

Search available versions:

helm search repo hashicorp/vault -l

Create Namespace & Install
kubectl create ns vault

helm install vault hashicorp/vault --namespace vault --version 0.34.1

Create ingress object
kubectl apply -f vault-ingress.yaml
### Initialize Vault

Open a shell in the Vault pod:

```bash
kubectl exec -it vault-0 -n vault -- sh

Inside the pod, run:

vault operator init
vault operator unseal

Critical: Save the unseal keys and root token securely. Do NOT commit to version control or Slack. Store in a secure password manager.

Step 8: Deploy Jenkins (Demo App - optional)
Create Namespace & Install
kubectl create ns jenkins
helm install my-jenkins jenkins/jenkins  -n jenkins

kubectl apply -f jenkins-ingress.yaml