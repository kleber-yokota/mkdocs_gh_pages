---
draft: false
date: 2024-03-26
categories:
  - Infrastructure
tags:
  - talos
  - k8s
  - kubernetes
  - homelab
  - tailscale
---

# Tailscale Operator

## **Motivation**

I’m setting up a homelab on a single machine, and ensuring it is both secure and part of the Tailscale network is crucial. The goal is to create an environment that closely resembles a production setup. By leveraging Tailscale, I can secure all the services in my homelab while maintaining a network that mimics the connectivity and security requirements of real-world production systems. This setup ensures that my infrastructure remains protected while providing the flexibility and scalability that I might need as the homelab evolves.

<!-- more -->

## Setup

### **Helm To install Helm**

To install Helm, run the following command:

```bash
brew install helm
```

Helm helps to install various components in Kubernetes, saving you from manually creating numerous YAML configuration files. You can add the Tailscale repository like this:

```sh
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
```

This command adds all necessary files to deploy Tailscale. Then, update the repository:

```
helm repo update
```

It's always a good idea to keep the files up to date.

## **Tailscale**

In the Tailscale admin console, add the following part to the `Access Control` section:

```json
"tagOwners": {
		"tag:k8s-operator": [],
		"tag:k8s":          ["tag:k8s-operator"],
	},
```

After that, go to `Settings` -> `OAuth Client` to generate an OAuth client with write access for `Devices Core` and `Auth Keys`. Add the tag `tag:k8s-operator` for both.

## **Installation**

### **Tailscale with Helm**

First, create the namespace and apply the configuration:

```bash
kubectl create namespace tailscale
kubectl label namespace tailscale pod-security.kubernetes.io/enforce=privileged
```

The first command creates the namespace, which is how Kubernetes organizes and applies group configurations. The second command enforces privileged mode in the namespace, which is required since Talos has strict security settings and doesn't allow certain privileges that Tailscale needs.

### **Why This Is Necessary:**

Talos enforces strict security policies, and for the Tailscale operator to function correctly, it requires certain privileges that are restricted in default configurations. The `pod-security.kubernetes.io/enforce=privileged` label ensures that the Tailscale operator has the necessary permissions to run.

Next, run the following command in the terminal, replacing the OAuth ID and secret provided by Tailscale:

```bash
helm upgrade \
  --install \
  tailscale-operator \
  tailscale/tailscale-operator \
  --namespace=tailscale \
  --create-namespace \
  --set-string oauth.clientId="<OAuth client ID>" \
  --set-string oauth.clientSecret="<OAuth client secret>" \
  --wait
```

### **Explanation:**

This command installs the Tailscale operator in the `tailscale` namespace. It configures the OAuth client ID and secret, which are necessary for authentication and authorization with Tailscale. The `--wait` flag ensures that Helm waits until the deployment is complete before returning.

Check if the `tailscale-operator` is connected on the machines.

## **Final Test: Verify NGINX IP in Tailscale**

Use the following configuration to expose the NGINX service with Tailscale:

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
  annotations:
    tailscale.com/expose: "true"
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Now apply the configuration:

```bash
kubectl create namespace demo-tailscale
kubectl apply -f tailscale_ingress.yaml  -n demo-tailscale
```

This configuration allows the Tailscale operator to detect the service and assign an IP within the Tailscale VPC, enabling secure communication across the network.

### **Key Point:**

Don’t forget to include this annotation in the service:

```yaml
annotations:
  tailscale.com/expose: "true"
```

This allows the Tailscale operator to detect the service and assign an IP within the Tailscale VPC, enabling secure communication across the network.
