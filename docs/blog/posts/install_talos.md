---
draft: false
date: 2024-03-12
categories:
  - Infrastructure
tags:
  - talos
  - k8s
  - kubernetes
  - homelab
---
# Talos - Standalone Installation on a Machine

## Creating a Bootable USB

Download the Talos ISO and create a bootable USB drive.
If you're unfamiliar with creating a bootable USB, you can use tools like [balenaEtcher](https://www.balena.io/etcher/) (Windows/macOS/Linux).
<!-- more -->
## Installing Talosctl

To install `talosctl`, run the following command:

```
brew install siderolabs/tap/talosctl
```

## Generating Configuration Files

Decide on a cluster name; in this case, we'll use `k8s-dev`.

Check the cluster's IP address or endpoint. In this example, we use `192.168.0.30`.

Run the following command to generate the configuration files:

```
talosctl gen config <cluster-name> https://<cluster-endpoint>:6443
```

For our example:

```
talosctl gen config k8s-dev https://192.168.0.30:6443
```

This will generate three files in the current directory: `controlplane.yaml`, `worker.yaml`, and `talosconfig`.

## Identifying the Installation Disk

To list the available disks on your machine, use the following command:

```sh
talosctl -n <cluster-endpoint> get disks --insecure
```

Example output:
![Pasted image 20250223190827.png](Pasted%20image%2020250223190827.png)

### How to Choose the Correct Disk

Each line represents a storage device available on the system. Here’s what to look for:

- **ID**: This is the device name you will use when modifying `controlplane.yaml` (e.g., `nvme0n1`, `sda`).
    
- **SIZE**: Check the disk size to ensure you're selecting the correct one.
    
- **TRANSPORT**: Identifies whether the disk is NVMe, USB, or another type.
    
- **ROTATIONAL**: If `true`, it's likely an HDD; if `false`, it's an SSD or NVMe.
    
- **MODEL**: Displays the manufacturer and model name of the disk.
    
- **READ ONLY**: If `true`, the disk cannot be written to.
    

> **Important:** Do not select the USB drive (e.g., `sda`) if you're installing Talos on an internal disk. Choose an NVMe or SATA SSD instead.

After determining the target disk, modify the `controlplane.yaml` file accordingly.

## Modifying controlplane.yaml

Locate the section containing the disk configuration:

![Pasted image 20250223164300.png](Pasted%20image%2020250223164300.png)

Change the disk path to `/dev/<your-chosen-disk>`. By default, Talos installs on the USB drive, but we want to specify a different disk.

- `image`: Defines the Talos version to install.
    
- `wipe`: If set to `true`, the disk will be formatted before installation.
    

For example, to install on `/dev/nvme0n1` with disk formatting enabled, update the configuration accordingly.

![Pasted image 20250223164827.png](Pasted%20image%2020250223164827.png)

Since using a single machine need to running workload on control-plane nodes

![Pasted image 20250223205336.png](Pasted%20image%2020250223205336.png)
Just uncomment the allowSchedulingOnControlPlanes

## Installing Talos

To apply the configuration and install Talos, run:

```
talosctl apply-config -n <cluster-endpoint> --file controlplane.yaml --insecure
```

For our example:

```
talosctl apply-config -n 192.168.0.30 --file controlplane.yaml --insecure
```

Wait for the API to become available:

```
talosctl -e <cluster-endpoint> -n <cluster-endpoint> containers
```

Example:

```
talosctl -e 192.168.0.30 -n 192.168.0.30 containers
```

Once confirmed, apply the bootstrap process:

```
talosctl bootstrap -n 192.168.0.30
```

## Verifying the Installation

To check the Talos services, run:

```
talosctl -e 192.168.0.30 -n 192.168.0.30 --talosconfig ./talosconfig services
```

A successful installation will show multiple running services, including `apid`, `containerd`, `etcd`, and `kubelet`.

## Testing Kubernetes

### Generating the Kubeconfig

Since this is the only Kubernetes cluster being used, generate the `kubeconfig` file with:

```
talosctl kubeconfig ./kubeconfig --nodes 192.168.0.30 --endpoints 192.168.0.30 --talosconfig=./talosconfig
```

Move the generated `kubeconfig` file to the appropriate location and test the cluster with:

```sh
kubectl get pods -A
```

### Testing Control Plane with a Simple Pod

To ensure the control plane is functioning properly, you can run a test pod with:
```sh
kubectl run test --restart=Never --image=hello-world
```
This will create a simple pod to verify that the Kubernetes control plane can deploy and manage workloads successfully.