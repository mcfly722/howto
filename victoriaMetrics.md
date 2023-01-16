# Deploy Victoria Metrics

### 1. Get API key to access CEPH Pool storage
```
ceph --cluster ceph auth get-key client.victoria-metrics-cluster
```
### 2. Create CEPH API Key secret
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: ceph-victoria-metrics-cluster-secret
  namespace: victoria-metrics-cluster
stringData:
  userID: victoria-metrics-cluster
  userKey: <BASE64 CEPH API Key>
EOF
```

### 3. Create Storage Class
```
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: ceph-victoria-metrics-cluster-storageclass
   namespace: victoria-metrics-cluster
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: <CEPH CLUSTER ID>
   pool: kub-sys-victoriametrics
   imageFeatures: layering
   csi.storage.k8s.io/provisioner-secret-name: ceph-victoria-metrics-cluster-secret
   csi.storage.k8s.io/provisioner-secret-namespace: victoria-metrics-cluster
   csi.storage.k8s.io/controller-expand-secret-name: ceph-victoria-metrics-cluster-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: victoria-metrics-cluster
   csi.storage.k8s.io/node-stage-secret-name: ceph-victoria-metrics-cluster-secret
   csi.storage.k8s.io/node-stage-secret-namespace: victoria-metrics-cluster
   csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
   - discard
EOF
```
### 4. Create PersistentVolumeClaim
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-victoria-metrics-cluster-claim
  namespace: victoria-metrics-cluster
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: <CEPH POOL SIZE IN GIGABYTES>Gi
  storageClassName: ceph-victoria-metrics-cluster-storageclass
EOF
```
### 5. Install Victoria Metrics Cluster
```
helm repo add vm https://victoriametrics.github.io/helm-charts

helm upgrade --install \
  --create-namespace \
  --namespace victoria-metrics-cluster \
  --set-json 'vmstorage.nodeSelector={"Datacenter":"APAC"}' \
  --set-json 'vmselect.nodeSelector={"Datacenter":"APAC"}' \
  --set vmselect.replicaCount=1 \
  --set vminsert.replicaCount=1 \
  --set vmstorage.replicaCount=1 \
  --set vmstorage.persistentVolume.enabled=true \
  --set vmstorage.persistentVolume.existingClaim="ceph-victoria-metrics-cluster-claim" \
  victoria-metrics-cluster vm/victoria-metrics-cluster
```
### 6. Specify where exactly Victoria Metrics Pods should be deployed
Could be specified using labels like this:
```
kubectl label nodes <your node name> Datacenter=APAC
kubectl get nodes --show-labels
```
