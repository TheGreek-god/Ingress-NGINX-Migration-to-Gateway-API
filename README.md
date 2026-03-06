# 🚀 Nginx Ingress to Gateway API Migration Project

## 📋 Project Overview
This project documents the complete journey of migrating from traditional NGINX Ingress to modern Gateway API using Traefik as the Gateway controller, with comprehensive auditing and validation tools.

## 🏗 Architecture

### Original Setup (NGINX Ingress)
- **Ingress Controller**: NGINX Ingress Controller
- **Application**: React app (zomato-clone) running on port 3000
- **Service**: ClusterIP on port 80
- **Domain**: zomato-app.com
- **TLS**: Self-signed certificate for HTTPS termination

## 📁 Project Structure

### Ingress-NGINX Migration to Gateway API
```
├── 📂 zomato-app/
│   ├── ingress.yaml                  # Original NGINX Ingress
│   ├── deployment.yaml               # React app deployment
|   ├── namespace.yaml                # React app service
│   └── service.yaml                  # ClusterIP service
├── 📂 traefik/
│   ├── gatewayclass.yaml             # GatewayClass definition
│   ├── gatewayapi.yaml               # Gateway resource with HTTP/HTTPS
│   ├── httproute-by-hostname.yaml    # HTTPRoute for routing
│   ├── middleware-headers.yaml       # Custom headers middleware
|   ├──middleware-sslredirect.yaml    # Middleware to redirect all HTTP traffic to HTTPS
│   └── values.yaml                   # Traefik Helm values
```

## 🔧 Prerequisites

- Kubernetes cluster (Minikube v1.36.0+)
- kubectl configured
- Helm v3+
- OpenSSL (for certificate generation)

## 📦 Installation Steps

1. **Start Minikube**

```bash
minikube start --driver=docker
minikube addons enable ingress
```
2. **Deploy the Application**
```bash
cd zomato-app
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f namespace.yaml
```
3. **Configure NGINX Ingress (Original)**

```bash
kubectl apply -f ingress.yaml
```
# Kubernetes Gateway API CRDs

The Kubernetes Gateway API is **not installed by default** on Kubernetes. It may become default at some point, but for now, we can install it from the Gateway API SIGs Guide.

> **Important Note:** At the time of this guide, we want to explore as many features as possible. Hence, the **experimental install** is used.

## Installing Gateway API

Run the following command to apply the experimental installation:

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/experimental-install.yaml
```
4. **Install Traefik with Gateway API Support**

```bash
cd traefik
helm repo add traefik https://helm.traefik.io/traefik
helm repo update

helm install traefik traefik/traefik \
  --version 39.0.2 \
  --values values.yaml \
  --namespace traefik \
  --create-namespace
  ```
  5. **Configure Gateway API Resources**

```bash
# Apply GatewayClass
kubectl apply -f gatewayclass.yaml

# Apply Gateway with HTTP/HTTPS listeners
kubectl apply -f gatewayapi.yaml

# Apply middleware for custom headers
kubectl apply -f middleware-headers.yaml

#Apply Middleware to redirect all HTTP traffic to HTTPS
kubectl apply -f middleware-sslredirect.yaml

# Apply HTTPRoute for routing
kubectl apply -f httproute-by-hostname.yaml

# Apply Traefik Custom Helm Values
kubectl apply -f values.yaml
```

6. **Generate TLS Certificate**

```bash
# Generate self-signed certificate (Git Bash on Windows)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout private.key \
  -out certificate.crt \
  -subj "//CN=zomato-app.com"

# Create Kubernetes TLS secret
kubectl create secret tls zomato-tls-secret \
  --cert=certificate.crt \
  --key=private.key \
  -n zomato-app
  ```

## 🔍 Verification & Auditing Tools

### NGINX Ingress Audit Function
Add this PowerShell function to your profile for quick ingress auditing:

```powershell
function Get-IngressAudit {
    kubectl get ingress -A -o json | ConvertFrom-Json | ForEach-Object { $_.items } | ForEach-Object { 
        [PSCustomObject]@{ 
            namespace = $_.metadata.namespace
            name = $_.metadata.name
            hosts = "zomato.com"
            migrated = if ($_.metadata.labels.migrated) { $_.metadata.labels.migrated } else { "<none>" }
            rewrite = if ($_.metadata.annotations.'nginx.ingress.kubernetes.io/rewrite-target') { $_.metadata.annotations.'nginx.ingress.kubernetes.io/rewrite-target' } else { "<none>" }
            ssl_redirect = if ($_.metadata.annotations.'nginx.ingress.kubernetes.io/ssl-redirect') { $_.metadata.annotations.'nginx.ingress.kubernetes.io/ssl-redirect' } else { "<none>" }
            snippet = if ($_.metadata.annotations.'nginx.ingress.kubernetes.io/configuration-snippet') { 
                $_.metadata.annotations.'nginx.ingress.kubernetes.io/configuration-snippet' -replace "`n", " " -replace "`r", ""
            } else { "<none>" }
            tls = if ($_.spec.tls.secretName) { ($_.spec.tls.secretName -join ',') } else { "<none>" }
        }
    } | Format-Table -Property @{Name="namespace";Expression={$_.namespace};Width=15},
        @{Name="name";Expression={$_.name};Width=20}, 
        @{Name="hosts";Expression={$_.hosts};Width=15}, 
        @{Name="migrated";Expression={$_.migrated};Width=10},
        @{Name="rewrite";Expression={$_.rewrite};Width=10},
        @{Name="ssl_redirect";Expression={$_.ssl_redirect};Width=12},
        @{Name="snippet";Expression={$_.snippet};Width=100},
        @{Name="tls";Expression={$_.tls};Width=20} -AutoSize
}

# Create alias for quick access
Set-Alias -Name iaudit -Value Get-IngressAudit

#Mark as done
kubectl label ingress zomato-app-ingress migrated=true -n zomato-app
```
## ✅ Testing & Validation

### Test NGINX Ingress

```bash
# Get NGINX service ports
minikube service ingress-nginx-controller -n ingress-nginx --url

# Test HTTP (should redirect to HTTPS)
curl -I -H "Host: zomato-app.com" http://127.0.0.1:PORT

# Test HTTPS with custom headers
curl -I -k -H "Host: zomato-app.com" https://127.0.0.1:PORT

# Verify TLS certificate
openssl s_client -connect 127.0.0.1:PORT -servername zomato-app.com -showcerts
```


### Test Traefik Gateway

```bash
# Get Traefik service ports
minikube service -n traefik traefik --url

# Test HTTP to HTTPS redirect
curl -I -H "Host: zomato-app.com" http://127.0.0.1:PORT

# Test HTTPS with custom headers
curl -vk -H "Host: zomato-app.com" https://127.0.0.1:PORT

# Verify TLS certificate
openssl s_client -connect 127.0.0.1:PORT -servername zomato-app.com -showcerts
```
## 📊 Migration Comparison

| Feature                    | NGINX Ingress                  | Gateway API (Traefik)          |
|----------------------------|--------------------------------|--------------------------------|
| **API Version**             | networking.k8s.io/v1           | gateway.networking.k8s.io/v1   |
| **Resource Types**          | Ingress                        | GatewayClass, Gateway, HTTPRoute |
| **TLS Termination**         | Ingress tls field              | Gateway listeners              |
| **Custom Headers**          | configuration-snippet          | Middleware resource            |
| **Path Rewriting**          | rewrite-target annotation      | URLRewrite filter              |
| **HTTP→HTTPS Redirect**     | ssl-redirect: "true"           | Gateway HTTPS listener         |
| **Namespace Isolation**     | Per Ingress                    | Gateway allowedRoutes policy   |

## 🐛 Common Issues & Solutions

### Issue 1: "Snippet directives are disabled"
**Solution**: Enable in ConfigMap

```bash
kubectl edit configmap -n ingress-nginx ingress-nginx-controller
# Add: allow-snippet-annotations: "true"
```
### Issue 2: Certificate not found

**Solution**: Generate and create secret
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout private.key -out certificate.crt \
  -subj "//CN=zomato-app.com"
kubectl create secret tls zomato-tls-secret \
  --cert=certificate.crt --key=private.key -n zomato-app
```

### Issue 3: Port 80/443 not accessible on Windows

**Solution**: Use high-numbered ports from minikube service
```bash
minikube service traefik -n traefik --url
# Use the provided ports 
```
### Issue 4: Gateway not accepting HTTPRoute

**Solution**: Check namespace alignment
```bash
# Ensure Gateway and HTTPRoute are in same namespace
kubectl get gateway -A
kubectl get httproute -A
```

## 🎯 Key Achievements

- ✅ **Dual Ingress Support**: Both NGINX and Traefik running simultaneously
- ✅ **TLS Termination**: Self-signed certificates working for both
- ✅ **HTTP→HTTPS Redirect**: Automatic redirects configured
- ✅ **Custom Headers**: X-Proxied-By and X-Proxy headers working
- ✅ **Path Rewriting**: / routes correctly transformed
- ✅ **Audit Tool**: Custom PowerShell function for quick verification
- ✅ **Migration Complete**: Feature parity between both implementations


