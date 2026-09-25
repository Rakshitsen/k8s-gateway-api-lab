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

