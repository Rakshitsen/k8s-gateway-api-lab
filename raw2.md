gcp gke Ingress Migration: Nginx Controller → Envoy Gateway API

Overview
This guide walks through migrating from Nginx Ingress Controller to Envoy Gateway—a modern, standards-based alternative implementing the Kubernetes Gateway API. Envoy Gateway offers:

Gateway API compliance: Uses standardized Gateway/HTTPRoute resources instead of legacy Ingress objects
Fine-grained traffic control: HTTPRoutes provide better routing semantics than Ingress
Advanced features: Traffic splitting, request/response modification, circuit breaking
Better observability: Native integration with Envoy Proxy's metrics and tracing
By the end of this section, you will have:

Envoy Gateway control plane running in the cluster
Gateway data plane pods managing traffic routing
Target group binding to ALB (via TargetGroupBinding)
HTTPRoutes configured for Vault and Jenkins applications
Updated security groups allowing ALB → Envoy traffic



Step 1: Install Envoy Gateway Control Plane

helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.9.1   -n envoy-gateway-system   --create-namespace



kubectl get pods -n envoy-gateway-system

Configure EnvoyProxy & GatewayClass
The EnvoyProxy custom resource defines how Envoy proxies behave. The GatewayClass tells Kubernetes which controller manages Gateway objects.


for to taget data plane pod to neg add we need annnoation at envoy proxy yaml configuration 
annotations:
          cloud.google.com/neg: '{"exposed_ports": {"80":{"name": "gateway-envoy-http-neg"}}}'


Create Gateway (Data Plane)
The Gateway resource provisions actual Envoy proxy pods that handle traffic routing. This is the data plane.


reate Gateway Manifest
Apply the Gateway:

kubectl apply -f gateway.yaml
3.2 Verify Data Plane Pods
Check that Envoy data plane pods are created:

kubectl get pods -n envoy-gateway-system
Expected output:

NAME                                                            READY   STATUS    RESTARTS   AGE
envoy-envoy-gateway-system-main-gateway-c3508b54-86b95d79d...   2/2     Running   0          30s
envoy-gateway-6fbfccc98d-q86rd                                  1/1     Running   0          5m
The pod name format is: envoy-{namespace}-{gateway-name}-{hash}.

Verify the data plane service is created:

kubectl get svc -n envoy-gateway-system
Expected output includes:

NAME                                         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
envoy-envoy-gateway-system-main-gateway      ClusterIP   10.100.50.XX   <none>        10080/TCP 15s

The service is type ClusterIP (not LoadBalancer) because gcp aLB via neg svcneg

at console after creation 

gateway-envoy-http-neg	Zonal NEG	1	Zonal (asia-south1-a)	mumbai-subnet	rakops-vpc-dev	
gateway-envoy-http-neg	Zonal NEG	0	Zonal (asia-south1-c)	mumbai-subnet	rakops-vpc-dev	
gateway-envoy-http-neg	Zonal NEG	0	Zonal (asia-south1-b)	mumbai-subnet	rakops-vpc-dev	



Create HTTPRoutes for Applications


HTTPRoutes define how traffic is routed to backend services. Replace Ingress objects with HTTPRoute.

5.1 Create Vault HTTPRoute
Apply the route:

kubectl apply -f vault-route.yaml
5.2 Create Jenkins HTTPRoute
Apply the route:

kubectl apply -f jenkins-route.yaml
Verify routes are bound:

kubectl get httproute -A
Expected output:

NAMESPACE   NAME             HOSTNAMES              AGE
vault       vault-route      [vault.rakops.in]      20s
jenkins     jenkins-route    [jenkins.rakops.in]    15s



Update firewall rules

gcloud compute firewall-rules create allow-envoy-ingress-from-lb \
    --network=rakops-vpc-dev \
    --action=ALLOW \
    --direction=INGRESS \
    --source-ranges=35.191.0.0/16,130.211.0.0/22 \
    --rules=tcp:10080,tcp:19003 \
    --target-tags=gke-rakops-cluster-fc8dabb5-node\
    --description="Allow Load Balancer traffic to Envoy on 10080 and health checks on 19003"


create health check 

gcloud compute health-checks create http envoy-proxy-health-check \
    --global \
    --port=19003 \
    --request-path=/ready \
    --check-interval=30s \
    --timeout=5s \
    --healthy-threshold=2 \
    --unhealthy-threshold=2 \
    --description="Health check for Envoy proxy on admin port 19003"




create a fresh new backend service 

gcloud compute backend-services create rakops-backend-envoy-service \
    --global \
    --load-balancing-scheme=EXTERNAL_MANAGED \
    --protocol=HTTP \
    --health-checks=envoy-proxy-health-check \
    --timeout=30s \
    --ip-address-selection-policy=IPV4_ONLY \
    --no-enable-cdn \
    --no-enable-logging




# Zone asia-south1-a
gcloud compute backend-services add-backend rakops-backend-envoy-service \
    --global \
    --network-endpoint-group=gateway-envoy-http-neg \
    --network-endpoint-group-zone=asia-south1-a \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=100 \
    --capacity-scaler=1.0

# Zone asia-south1-b
gcloud compute backend-services add-backend rakops-backend-envoy-service  \
    --global \
    --network-endpoint-group=gateway-envoy-http-neg\
    --network-endpoint-group-zone=asia-south1-b \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=100 \
    --capacity-scaler=1.0

# Zone asia-south1-c
gcloud compute backend-services add-backend rakops-backend-envoy-service  \
    --global \
    --network-endpoint-group=gateway-envoy-http-neg \
    --network-endpoint-group-zone=asia-south1-c \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=100 \
    --capacity-scaler=1.0



update backend service from rakops-backend-service	Backend service to  rakops-backend-envoy-service	Backend service



verify everything is working
curl -I https://jenkins.rakops.in
curl -I https://vault.rakops.in