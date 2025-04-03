---
draft: false
date: 2024-04-03
categories:
  - Infrastructure
tags:
  - talos
  - k8s
  - kubernetes
  - homelab
  - tailscale
  - traefik
---


# Traefik
Traefik is a reverse proxy, an alternative to Nginx. I found it interesting and decided to use it for my home lab. But why use a reverse proxy when Tailscale already provides a built-in DNS for devices connected to the network? The problem is that the assigned hostname is random and ends with `.ts.net`, making it difficult to remember. Additionally, remembering the IP addresses of applications is not practical.

Since Tailscale offers MagicDNS, which allows all connected devices to share a common domain and subdomain, I can leverage my own DNS and use a reverse proxy to simplify access. This setup also enables me to obtain TLS certificates for better security.

## Installing Traefik
<!-- more -->
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

Create a file named `values.yaml`. This file overrides the default chart values to configure Traefik as needed.

```yaml
service:  
  type: LoadBalancer  
  spec:  
    loadBalancerClass: tailscale
    
ports:  
  web:  
    redirectTo:  
      port: websecure
```

To deploy Traefik:

```bash
helm install traefik traefik/traefik -n traefik --create-namespace --values values.yaml
```

This will ensure that the Traefik service gets an IP address within the Tailscale network. You can then check which IP address has been assigned.

### Testing Traefik with Nginx

To test if Traefik is working, I'll deploy an Nginx instance:

```yaml
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: nginx  
  labels:  
    app: nginx  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      app: nginx  
  template:  
    metadata:  
      labels:  
        app: nginx  
    spec:  
      containers:  
        - name: nginx  
          image: nginx:latest  
          ports:  
            - containerPort: 80  

---  
apiVersion: v1  
kind: Service  
metadata:  
  name: nginx  

spec:  
  type: ClusterIP  
  selector:  
    app: nginx  
  ports:  
    - port: 80  
      targetPort: 80  

---  
apiVersion: traefik.io/v1alpha1  
kind: IngressRoute  
metadata:  
  name: nginx-ingress  
  namespace: default  
spec:  
  entryPoints:  
    - websecure  
  routes:  
    - match: Host(`nginx.upliftdata.pro`)  
      kind: Rule  
      services:  
        - name: nginx  
          port: 80  
```

When connected to the network, you can use the Tailscale-provided URL to access the service.

## Cloudflare Configuration

In Cloudflare's DNS settings, go to your domain, navigate to DNS, and set the proxy status to **DNS only**. Add the subdomain and the Tailscale IP assigned to the Traefik service. You can find this IP in the Tailscale dashboard.

## Cert-Manager for TLS Certificates

Since I have my own domain managed via Cloudflare, I can use it within Tailscale. I will also use Cloudflare for DNS and Let's Encrypt for issuing TLS certificates.

To generate TLS certificates, Cloudflare requires an API token with the following permissions:
- **Zone: Read**
- **DNS: Edit**

### Installing Cert-Manager

First, add the Cert-Manager Helm repository:

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
```

Then, create a `cert_manager_values.yaml` file:

```yaml
namespace: cert-manager  
crds:  
  enabled: true  
extraArgs:  
  - --dns01-recursive-nameservers-only  
  - --dns01-recursive-nameservers=1.1.1.1:53,1.0.0.1:53
```

Install Cert-Manager using Helm:

```bash
helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --values cert_manager_values.yaml
```

### Creating the Secret for Cloudflare API Token

Remember to encode the token in **Base64** before adding it to the secret:

```yaml
apiVersion: v1  
kind: Secret  
metadata:  
    name: cloudflare-api-key-secret  
    namespace: cert-manager  
type: Opaque  
data:  
    api-key: <base64_encoded_token>
```

### Configuring a ClusterIssuer for Let's Encrypt

The `ClusterIssuer` defines how certificates will be issued using Let's Encrypt:

```yaml
apiVersion: cert-manager.io/v1  
kind: ClusterIssuer  
metadata:  
  name: cloudflare-issuer  
  namespace: cert-manager  
spec:  
  acme:  
    server: https://acme-v02.api.letsencrypt.org/directory  
    email: <your_cloudflare_email>  
    privateKeySecretRef:  
      name: cloudflare-issuer-account-key  
    solvers:  
    - dns01:  
        cloudflare:  
          email: <your_cloudflare_email>  
          apiTokenSecretRef:  
            name: cloudflare-api-key-secret  
            key: api-key
```

This configuration points to the **production** Let's Encrypt environment. If needed, a separate `ClusterIssuer` for the **staging** environment can be created for testing purposes.

### Issuing a TLS Certificate

To generate the actual TLS certificate, create a `Certificate` resource:

```yaml
apiVersion: cert-manager.io/v1  
kind: Certificate  
metadata:  
  name: nginx-certificate  
  namespace: default  
spec:  
  secretName: nginx-certificate-prod  
  issuerRef:  
    name: cloudflare-issuer  
    kind: ClusterIssuer  
  dnsNames:  
    - <your_domain_ex: nginx.example.com>
```

### Updating the IngressRoute for TLS

Now, update the `IngressRoute` to use the generated certificate:

```yaml
apiVersion: traefik.io/v1alpha1  
kind: IngressRoute  
metadata:  
  name: nginx-ingress  
  namespace: default  
spec:  
  entryPoints:  
    - websecure  
  routes:  
    - match: Host(`nginx.upliftdata.pro`)  
      kind: Rule  
      services:  
        - name: nginx  
          port: 80  
  tls:  
    secretName: nginx-certificate-prod
```

This ensures that the Nginx service is accessible securely using a TLS certificate issued via Let's Encrypt, managed by Cert-Manager, and proxied through Traefik.

