---
draft: false
date: 2024-05-22
categories:
  - Infrastructure
  - Datastack
tags:
  - talos
  - k8s
  - kubernetes
  - homelab
  - datastack
  - clickhouse
---
# ClickHouse on Kubernetes in My Homelab

## What is ClickHouse?

**ClickHouse** is a columnar database, highly performant and widely used for **analytics** and handling large datasets. It serves as the **Data Warehouse** in my homelab.

To deploy ClickHouse on my Kubernetes cluster, I use the **Altinity** distribution, which provides an **Operator** and Kubernetes **CRDs (Custom Resource Definitions)**. This Operator adds custom resources to manage ClickHouse, like the `chi` command to check cluster status.

---

## Installing the ClickHouse Operator

### Step 1 — Install the Operator

Run the following command to install the Altinity Operator on Kubernetes:
<!-- more -->
```bash
kubectl apply -f https://raw.githubusercontent.com/Altinity/clickhouse-operator/master/deploy/operator/clickhouse-operator-install-bundle.yaml
```

### Step 2 — Create a Namespace

```bash
kubectl create namespace clickhouse-demo
```

---

## Deploying ClickHouse

Below is the YAML I use to deploy a simple ClickHouse cluster in my homelab with:

- **3 shards** (data partitions);
- **1 replica** per shard.

### Important notes:

- The `users` section defines the username and password. If you don’t specify network access (`networks/ip`), it may block external connections.
- This YAML is integrated with **Longhorn** as the storage backend, so volumes are automatically provisioned.

```yaml
apiVersion: clickhouse.altinity.com/v1
kind: ClickHouseInstallation
metadata:
  name: clickhouse-demo
  namespace: clickhouse-demo
spec:
  defaults:
    templates:
      podTemplate: clickhouse-pod
      volumeClaimTemplate: clickhouse-volume
  configuration:
    users:
      default/password: qwerty
      default/networks/ip: "::/0"
    clusters:
      - name: cluster1
        layout:
          shardsCount: 3
          replicasCount: 1
        templates:
          podTemplate: clickhouse-pod
          volumeClaimTemplate: clickhouse-volume
  templates:
    podTemplates:
      - name: clickhouse-pod
        spec:
          containers:
            - name: clickhouse
              image: clickhouse/clickhouse-server:25.4
    serviceTemplates:
      - name: cluster1-cluster-ip
        spec:
          type: ClusterIP
          ports:
            - name: http
              port: 8123
              targetPort: 8123
            - name: https
              port: 8443
              targetPort: 8443
            - name: tcp
              port: 9000
              targetPort: 9000
    volumeClaimTemplates:
      - name: clickhouse-volume
        spec:
          accessModes: [ "ReadWriteOnce" ]
          storageClassName: longhorn
          resources:
            requests:
              storage: 100Gi
```

---

## Creating a Manual Service

Although the Operator creates some services automatically, I prefer creating a manual Service for better control, especially when exposing it via **Traefik** and securing with **Let's Encrypt**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clickhouse-clusterip
  namespace: clickhouse-demo
spec:
  selector:
    clickhouse.altinity.com/chi: clickhouse-demo
  ports:
    - name: http
      port: 8123
      targetPort: 8123
    - name: tcp
      port: 9000
      targetPort: 9000
  type: ClusterIP
```

---

## Exposing with Traefik and TLS

In my setup, I use **Traefik** as the Ingress Controller with **Let's Encrypt** for TLS certificates. For this, I create:

- An `IngressRoute` in Traefik;
- A `Certificate` with cert-manager.

> ⚠️ **Important:** Ensure your DNS points to your cluster IP, and Traefik is configured to handle the domain.

---

## Testing the Connection

You can test the connection to ClickHouse using Python with the `clickhouse-connect` library.

### Install the library:

```bash
pip install clickhouse-connect
```

### Test script:

```python
from clickhouse_connect import get_client

client = get_client(
    host='clickhouse.upliftdata.pro',
    port=443,
    username='default',
    password='qwerty',
    secure=True
)

result = client.query('SELECT now()').result_rows
print(result)
```

The expected output is the **current datetime**, confirming the connection works.

---

## Conclusion

With this setup, ClickHouse is running in my Kubernetes cluster with high availability, persistent storage via Longhorn, and secure exposure using Traefik. This is one more production-grade component running in my homelab.
