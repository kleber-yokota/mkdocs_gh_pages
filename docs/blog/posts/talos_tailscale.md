---
draft: false
date: 2024-03-20
categories:
  - Infrastructure
tags:
  - talos
  - k8s
  - kubernetes
  - homelab
  - tailscale
---

# Talos - Adding Features and Tips

## **What is a VPC and Why Companies Use It?**

A **Virtual Private Cloud (VPC)** is an isolated, private network within a public cloud provider or an on-premises environment. It allows secure communication between resources without exposing them to the public internet.

### **Why Companies Use VPCs?**

- **Security:** Resources inside a VPC are **isolated** and protected from external access.
    
- **Private Networking:** Services can communicate **internally** without needing public IPs.
    
- **Scalability:** Easily expand the infrastructure while maintaining security policies.
    
- **Remote Access:** With the right setup (e.g., VPN or Tailscale), teams can securely manage internal services.
    

### **Why Use a VPC in My Homelab?**

In my homelab, I aim to create an environment that closely resembles a production setup. Since I’m often on the go, I need **reliable and secure remote access** to manage my infrastructure from anywhere.

Instead of exposing public IPs—which **increases security risks** like unauthorized access and attacks—I use **Tailscale** to build a secure, encrypted **VPC over the internet**. This allows me to connect to my Talos cluster seamlessly, just as if I were on the local network, while keeping everything private and protected.

---

## **Adding Tailscale Access to the Cluster**

Talos supports extensions, and one of them is Tailscale. With this, we can create a VPC and access the cluster remotely.

To add a new capability, you need to create a custom Talos image with Tailscale using [Talos Factory](https://factory.talos.dev/). Once generated, the factory provides an image ID that you will use to install and upgrade Talos.

![Pasted image 20250224202046.png](Pasted%20image%2020250224202046.png)

This new image will be used to upgrade Talos.

![Pasted image 20250224203809.png](Pasted%20image%2020250224203809.png)

### **Upgrade Talos with the New Image**

Run the following command to upgrade Talos with the new image:

```
talosctl upgrade --image factory.talos.dev/installer/4a0d65c669d46663f377e7161e50cfd570c401f26fd9e7bda34a0216b6f1922b:v1.9.4 -m powercycle -f -e 192.168.0.30 -n 192.168.0.30
```

### **Verify Tailscale Extension Installation**

To check if the Tailscale extension is installed, use:

```
talosctl -e 192.168.0.30 -n 192.168.0.30 get extensions
```

Expected output:

```
NODE           NAMESPACE   TYPE              ID   VERSION   NAME        VERSION
192.168.0.30   runtime     ExtensionStatus   0    1         tailscale   1.78.1
192.168.0.30   runtime     ExtensionStatus   1    1         schematic   xxxxxxxxxxxxxxxxxxxxxxxxx
```

### **Generate and Apply Tailscale Auth Key**

In Tailscale, generate an authentication key. Then, add it to `tailscale.patch.yaml`:

```
---
apiVersion: v1alpha1
kind: ExtensionServiceConfig
name: tailscale
environment:
  - TS_AUTHKEY=<tailscale auth key>
```

### **Enable Tailscale**

Apply the configuration:

```
talosctl apply-config -n 192.168.0.30 -e 192.168.0.30 --file controlplane.yaml -p @tailscale.patch.yaml
```

Check the Tailscale admin panel to verify the cluster’s IP.

### **Update Control Plane Configuration**

To allow Talos to receive commands via Tailscale, update `certSANs` with the Tailscale IP and reapply the configuration:

```
talosctl apply-config -n 192.168.0.30 -e 192.168.0.30 --file controlplane.yaml -p @tailscale.patch.yaml
```

---

## **Tips**

### **1. Setting Nodes and Endpoints**

Typing the endpoint and nodes every time for `talosctl` can be tedious. If you have two IPs (one local and one from Tailscale), you can store them in the Talos config:

```
talosctl config endpoint 192.168.0.30
talosctl config nodes 192.168.0.30
```

However, running this command again will **overwrite** the previous configuration instead of adding the Tailscale IP. To keep both the local and Tailscale IPs, you need to **manually edit** the Talos config file and add the Tailscale IP under `endpoints` and `nodes`:

```
contexts:
    k8s-dev:
        endpoints:
            - 192.168.0.30
            - <Tailscale IP>
        nodes:
            - 192.168.0.30
            - <Tailscale IP>
```

### **2. Applying Configurations Properly**

If you forget to include `-p @tailscale.patch.yaml` in the `apply-config` command, the control plane won’t merge with the Tailscale configuration, leaving your cluster offline in the Tailscale network.

Also, ensure you use `---` in YAML files to separate and merge multiple configurations correctly.

### **3. Updating** `**kubectl**` **Config**

Update your `kubectl` config to use the Tailscale IP instead of the local IP.