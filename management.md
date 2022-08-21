# Install K8S Management Tools (Dashboard)
## install Dashboard
(https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard )

### issue Dashboard Web certificate
```
openssl req -x509 -nodes \
 -newkey rsa:4096 \
 -CA rootCA.crt \
 -CAkey rootCA.key \
 -days 365000 \
 -subj '/CN=dashboard.59ff44dd.nip.io' \
 -addext 'extendedKeyUsage=1.3.6.1.5.5.7.3.1' \
 -addext 'keyUsage=keyEncipherment' \
 -keyout dashboard-web.key \
 -out dashboard-web.crt
```
### install Dashboard
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

helm install \
 --namespace dashboard-system \
 --create-namespace \
kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard
```
### add Dashboard web certificate to k8s secret
```
kubectl create secret tls dashboard-web-tls \
  --namespace dashboard-system \
  --key dashboard-web.key \
  --cert dashboard-web.crt
```
### create Ingress rule for Dashboard
```
cat <<EOT > dashboard-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard
  namespace: dashboard-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.org/ssl-services: "kubernetes-dashboard"
spec:
  rules:
  - host: "dashboard.59ff44dd.nip.io"
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443
  tls:
    - hosts:
      - dashboard.59ff44dd.nip.io
      secretName: dashboard-web-tls
EOT
```
```
kubectl apply -f dashboard-ingress.yaml
```
---
## Create K8S Access Token

### create k8s account
```
kubectl create serviceaccount mcfly722
```
### create k8s account role binding
```
kubectl create clusterrolebinding mcfly722 -n default --clusterrole=cluster-admin --serviceaccount=default:mcfly722
```
### get k8s account token

```
kubectl get secrets -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='mcfly722')].data.token}"|base64 --decode
```
now you can use it to authenticate on dashboard.<br>

## Configure kubectl config access

### issue kubernetes API certificate
```
openssl req -x509 -nodes \
 -newkey rsa:4096 \
 -CA rootCA.crt \
 -CAkey rootCA.key \
 -days 365000 \
 -subj '/CN=kubernetes.59ff44dd.nip.io' \
 -addext 'extendedKeyUsage=1.3.6.1.5.5.7.3.1' \
 -addext 'keyUsage=keyEncipherment' \
 -keyout kubernetes-web.key \
 -out kubernetes-web.crt
```
### save API certificate to secret 
```
kubectl create secret tls kubernetes-api-tls \
  --namespace default \
  --key kubernetes-web.key \
  --cert kubernetes-web.crt
```
### create Ingress publication
```
cat <<EOT > kubernetes-api-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.org/ssl-services: "kubernetes"
spec:
  rules:
  - host: "kubernetes.59ff44dd.nip.io"
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
            name: kubernetes
            port:
              number: 443
  tls:
    - hosts:
      - kubernetes.59ff44dd.nip.io
      secretName: kubernetes-api-tls
EOT
```
```
kubectl apply -f kubernetes-api-ingress.yaml
```
### create %PROFILE%/.kube/config
```
kubectl config set-cluster default --server='https://kubernetes.59ff44dd.nip.io' --insecure-skip-tls-verify=true
kubectl config set-credentials default --token=<YOUR TOKEN>
kubectl config set-context default --cluster default --user=default
kubectl config use-context default
```
