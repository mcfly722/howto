# CEPH Scenarios

### 1. Obtain CEPH cluster ID and Monitors IP's
```
ceph mon dump
```
This command shows Cluster ID (fsid) and all monitors IPs
### 2. Install K8S Ceph plugin

```
helm repo add ceph-csi https://ceph.github.io/csi-charts

helm upgrade --install \
--namespace kube-system \
--set-json 'csiConfig=[
    {
        "clusterID": "<CEPH Cluster1 ID>",
        "monitors": [
        "v2:<CEPH OSD1 IP>:3300/0,v1:<CEPH OSD1 IP>:6789/0",
        "v2:<CEPH OSD2 IP>:3300/0,v1:<CEPH OSD2 IP>:6789/0",
        "v2:<CEPH OSD3 IP>:3300/0,v1:<CEPH OSD3 IP>:6789/0"
        ]
    },
    {
        "clusterID": "<CEPH Cluster2 ID>",
        "monitors": [
        "v2:<CEPH OSD1 IP>:3300/0,v1:<CEPH OSD1 IP>:6789/0",
        "v2:<CEPH OSD2 IP>:3300/0,v1:<CEPH OSD2 IP>:6789/0",
        "v2:<CEPH OSD3 IP>:3300/0,v1:<CEPH OSD3 IP>:6789/0"
        ]
    }
]' \
ceph-csi-rbd ceph-csi/ceph-csi-rbd
```

### 3. Generate API key to access to CEPH pool

#### 3.1 Create new access account
```
ceph auth get-or-create client.<POOL NAME> mon 'allow r' osd 'allow rwx pool=<POOL NAME>'
```
#### 3.2 Get account API key
```
ceph --cluster ceph auth get-key client.<POOL NAME>
```
### 4. Add secret key to K8S Cluster in required namespace
```
kubectl create secret generic ceph-secret-<POOL NAME> \
  --namespace=<NAMESPACE> \
  --type="kubernetes.io/rbd" \
  --from-literal=key='base64 encoded CEPH API key'
```
### 5. Create Storage Class
