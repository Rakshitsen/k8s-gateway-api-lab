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