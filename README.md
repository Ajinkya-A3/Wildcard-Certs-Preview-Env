# Wildcard TLS + Preview Environments using Kubernetes Gateway API

## Overview

This setup enables **dynamic preview environments using wildcard
subdomains** in Kubernetes.

Example preview URLs:

    https://test1.example.com
    https://test2.example.com
    https://feature-login.example.com

All preview URLs share a **single wildcard TLS certificate** issued by
Let's Encrypt using **DNS01 validation via Cloudflare**.

### Traffic Flow

    Internet
       ↓
    Cloudflare DNS (*.example.com)
       ↓
    GKE Gateway Load Balancer
       ↓
    Gateway API
       ↓
    HTTPRoute
       ↓
    Service
       ↓
    Pod

------------------------------------------------------------------------

# Architecture

    *.example.com
          │
          ▼
    Cloudflare DNS
          │
          ▼
    GKE Gateway (External Load Balancer)
          │
          ▼
    HTTPRoute
          │
          ▼
    Service
          │
          ▼
    Preview Application Pod

------------------------------------------------------------------------

# Prerequisites

## 1. Kubernetes Cluster

Cluster must support **Gateway API**.

Example:

-   Google Kubernetes Engine (GKE)
-   Gateway Class:


```
    gke-l7-global-external-managed

```

------------------------------------------------------------------------

## 2. Install cert-manager

Install cert-manager in the cluster:

    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

------------------------------------------------------------------------

## 3. Cloudflare DNS

Create a wildcard DNS record.
```

  Type   Name   Value
  ------ ------ ------------
  A       *     GATEWAY_IP
```

Example:

    *.example.com → 34.xxx.xxx.xxx

------------------------------------------------------------------------

## 4. Cloudflare API Token

Create an API token with permissions:

    Zone → DNS → Edit
    Zone → Zone → Read

Create Kubernetes secret:

    kubectl create secret generic cloudflare-api-token-secret --from-literal=api-token=YOUR_TOKEN -n cert-manager

------------------------------------------------------------------------

# Project Structure

    K8s/
     ├── clusterissuer.yaml
     ├── wildcard-certificate.yaml
     ├── gateway.yaml
     ├── httproute.yaml
     ├── dep.yaml
     └── svc.yaml

------------------------------------------------------------------------

# Step 1 --- Create ClusterIssuer

    kubectl apply -f K8s/clusterissuer.yaml

------------------------------------------------------------------------

# Step 2 --- Create Wildcard Certificate

This issues a certificate for:

    *.example.com

Secret created:

    wildcard-example-tls

Apply:

    kubectl apply -f K8s/wildcard-certificate.yaml

Verify:

    kubectl get certificate -n gateway-system

Expected:

    READY: True

------------------------------------------------------------------------

# Step 3 --- Create Gateway

    kubectl apply -f K8s/gateway.yaml

Get Gateway IP:

    kubectl get gateway wildcard-gateway -n gateway-system

------------------------------------------------------------------------

# Step 4 --- Deploy Test Application

    kubectl apply -f K8s/dep.yaml

The test container returns:

    Hello from wildcard test app

------------------------------------------------------------------------

# Step 5 --- Create Service

    kubectl apply -f K8s/svc.yaml

Verify:

    kubectl get svc

------------------------------------------------------------------------

# Step 6 --- Create HTTPRoute

    kubectl apply -f K8s/httproute.yaml

Verify:

    kubectl get httproute

------------------------------------------------------------------------

# Step 7 --- Test DNS

    nslookup test1.example.com

Expected:

    GATEWAY_IP

------------------------------------------------------------------------

# Step 8 --- Test HTTP

    curl http://test1.example.com

Expected:

    Hello from wildcard test app

------------------------------------------------------------------------

# Step 9 --- Test HTTPS

    curl -v https://test1.example.com

Expected TLS:

    issuer: Let's Encrypt
    subject: *.example.com

Response:

    Hello from wildcard test app

------------------------------------------------------------------------

# Debugging

Check certificates:

    kubectl get certificate -A

Check gateway:

    kubectl describe gateway wildcard-gateway -n gateway-system

Check routes:

    kubectl describe httproute wildcard-route

Check pods:

    kubectl get pods

------------------------------------------------------------------------

# Preview Environment Example

Your CI/CD pipeline can create preview URLs automatically:

    feature-login.example.com
    pr-101.example.com
    preview-auth.example.com

No DNS changes are required because:

    *.example.com

handles all subdomains.

------------------------------------------------------------------------

# Advantages

-   Single wildcard certificate
-   Unlimited preview environments
-   No DNS changes required
-   Automatic TLS using cert-manager
-   Scales to thousands of subdomains
