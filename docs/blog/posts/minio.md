---
draft: false
date: 2024-04-10
categories:
  - Infrastructure
  - Datastack
tags:
  - talos
  - k8s
  - kubernetes
  - homelab
  - tailscale
  - datastack
  - minio
  - longhorn
---
## MinIO + Longhorn + Traefik + Cert-Manager + Cloudflare on Talos Kubernetes

### What is MinIO?

MinIO is an object storage system, just like AWS S3. In fact, it uses the same protocol. This means you can store any kind of file and access it via APIs or SDKs — a great foundation for your data stack.

It makes a lot of sense to have an object store in your infrastructure.

---

<!-- more -->
### How to persist data

Kubernetes runs stateless containers — which means pods don’t keep any data. Every time a pod is restarted or scaled, it starts fresh.

But apps like MinIO or databases **need** to persist data. If you run a database pod without any volume, it will boot up empty every time.

To solve this, you create a `PersistentVolume` and a `PersistentVolumeClaim`. This way, your app knows where to store and recover data even after restarts.

In this setup, I’ll run everything on a single node, so I won’t worry about complex scheduling.

---

### Installing Longhorn

To manage storage, I’m using **Longhorn**, which automatically provisions volumes.

Longhorn needs special privileges, so let’s isolate it:

```bash
kubectl create namespace longhorn-system
kubectl label namespace longhorn-system pod-security.kubernetes.io/enforce=privileged
```

Then, configure **Talos OS** to allow disk operations. This requires enabling two extensions and mounting a folder Longhorn uses.

Use [Talos Factory](https://factory.talos.dev/) to build a custom image with these extensions:

- `util-linux-tools`
- `iscsi-tools`

```bash
talos upgrade --image factory.talos.dev/installer/<IMAGE_HASH>:v1.9.4 -m powercycle -f -e 192.168.0.30 -n 192.168.0.30
```

> Warning: use only one IP for both `-e` and `-n`.

Now update your `talosconfig`:

```yaml
machine:
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options:
          - bind
          - rshared
          - rw
```

Apply it:

```bash
talosctl apply-config -f controlplane.yaml
```

---

### Installing Longhorn via Helm

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

Create a `longhorn_values.yaml`:

```yaml
defaultSettings:
  defaultReplicaCount: 1
persistence:
  reclaimPolicy: Retain
  defaultClassReplicaCount: 1
```

Then install:

```bash
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.8.1 -f longhorn_values.yaml
```

---

### Setting up DNS, TLS certificate, and Ingress (Traefik + cert-manager + Cloudflare)

You’ll expose Longhorn with a domain like `longhorn.<your-domain>` pointing to your Traefik Load Balancer IP.

#### Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: longhorn-certificate
  namespace: longhorn-system
spec:
  secretName: longhorn-certificate-prod
  issuerRef:
    name: cloudflare-issuer-prod
    kind: ClusterIssuer
  dnsNames:
    - longhorn.<your-domain>
```

> Wait for the certificate to be ready before applying the IngressRoute.

#### IngressRoute

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`longhorn.<your-domain>`)
      kind: Rule
      services:
        - name: longhorn-frontend
          port: 80
  tls:
    secretName: longhorn-certificate-prod
```

---

### Installing MinIO

```bash
helm repo add minio https://operator.min.io/
helm repo update
```

Create a `values.yaml` for the operator:

```yaml
operator:
  env:
    - name: OPERATOR_STS_ENABLED
      value: "on"
  replicaCount: 1
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
      ephemeral-storage: 500Mi
```

Then install the operator:

```bash
helm install minio minio-operator/operator -n minio -f values.yaml
```

Now install a **tenant**, which is the actual MinIO instance.

Create `tenant.yaml`:

```yaml
tenant:
  name: myminio
  image:
    repository: quay.io/minio/minio
    tag: RELEASE.2025-03-12T18-04-18Z
    pullPolicy: IfNotPresent
  configSecret:
    name: myminio-env-configuration
    accessKey: minio
    secretKey: minio123
  pools:
    - servers: 1
      name: pool-0
      volumesPerServer: 1
      size: 300Gi
      storageClassName: longhorn
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        fsGroupChangePolicy: "OnRootMismatch"
        runAsNonRoot: true
      containerSecurityContext:
        runAsUser: 1000
        runAsGroup: 1000
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
        seccompProfile:
          type: RuntimeDefault
  mountPath: /export
  subPath: /data
  certificate:
    requestAutoCert: false
  exposeServices:
    console: false
    minio: false
```

> ⚠️ Credentials are hardcoded for now. You should extract them into a Kubernetes Secret in production.

```bash
helm install --namespace tenant-ns --create-namespace tenant minio/tenant -f tenant.yaml
```

Note: MinIO tries to generate a self-signed cert which can break health checks. We’ll fix that using cert-manager.

#### Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: minio-tenant-certificate
  namespace: tenant-ns
spec:
  secretName: minio-tenant-certificate-prod
  issuerRef:
    name: cloudflare-issuer-prod
    kind: ClusterIssuer
  dnsNames:
    - minio.<your-domain>
```

#### IngressRoute

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: minio-tenant-ingress
  namespace: tenant-ns
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`minio.<your-domain>`)
      kind: Rule
      services:
        - name: myminio-console
          port: 9090
  tls:
    secretName: minio-tenant-certificate-prod
```

---

### Bonus tip: using multiple NVMe disks with Longhorn

If you have multiple NVMe drives, you can configure Talos to use all of them with Longhorn.

List all available disks:

```bash
talosctl get DiscoveredVolume
```

Then add this to your Talos config:

```yaml
machine:
  disks:
    - device: /dev/nvme2n1
      partitions:
        - mountpoint: /var/lib/longhorn/nvme2n1/
    - device: /dev/nvme3n1
      partitions:
        - mountpoint: /var/lib/longhorn/nvme3n1/
```

Apply and reboot:

```bash
talosctl apply-config -f controlplane.yaml
talosctl reboot
```

Now go to the Longhorn dashboard and add the new disks to each node.

---

Enjoy your highly available object store setup with MinIO + Longhorn on Kubernetes!
