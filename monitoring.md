# Configuring K8S Cluster Monitoring
## install Prometheus
https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus/values.yaml
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable

helm upgrade --install \
--namespace prometheus-system \
--create-namespace \
--set alertmanager.enabled=false \
--set pushgateway.enabled=false \
--set kubeStateMetrics.enabled=false \
--set nodeExporter.enabled=false \
--set server.extraFlags[0]=storage.tsdb.retention.size=900MB \
--set server.persistentVolume.size=1Gi \
prometheus-system prometheus-community/prometheus
```
### issue Prometheus web certificate
```
openssl req -x509 -nodes \
 -newkey rsa:4096 \
 -CA rootCA.crt \
 -CAkey rootCA.key \
 -days 365000 \
 -subj '/CN=prometheus.59ff44dd.nip.io' \
 -addext 'extendedKeyUsage=1.3.6.1.5.5.7.3.1' \
 -addext 'keyUsage=keyEncipherment' \
 -keyout prometheus-web.key \
 -out prometheus-web.crt
```
### save Prometheus web certificate to secret 
```
kubectl create secret tls prometheus-web-tls \
  --namespace prometheus-system \
  --key prometheus-web.key \
  --cert prometheus-web.crt
```
### create Prometheus Ingress publication
```
cat <<EOT > prometheus-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  namespace: prometheus-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: "prometheus.59ff44dd.nip.io"
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
            name: prometheus-system-server
            port:
              number: 80
  tls:
    - hosts:
      - prometheus.59ff44dd.nip.io
      secretName: prometheus-web-tls
EOT
```
```
kubectl apply -f prometheus-ingress.yaml
```
## install Prometheus K8S Node Exporter
```
helm upgrade --install \
--namespace prometheus-system \
--create-namespace \
prometheus-node-exporter prometheus-community/prometheus-node-exporter
```
## install Grafana
https://github.com/grafana/helm-charts/blob/main/charts/grafana/README.md
### add Grafana repo
```
helm repo add grafana https://grafana.github.io/helm-charts
```

### prepare Grafana values
```
cat <<EOT > grafana-values.yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-system-server.prometheus-system.svc.cluster.local
EOT
```
### install Grafana
```
helm upgrade --install \
--namespace grafana-system \
--create-namespace \
--set adminPassword=<GRAFANA SITE PASSWORD> \
-f grafana-values.yaml \
grafana-system grafana/grafana
```
### issue Grafana web certificate
```
openssl req -x509 -nodes \
 -newkey rsa:4096 \
 -CA rootCA.crt \
 -CAkey rootCA.key \
 -days 365000 \
 -subj '/CN=grafana.59ff44dd.nip.io' \
 -addext 'extendedKeyUsage=1.3.6.1.5.5.7.3.1' \
 -addext 'keyUsage=keyEncipherment' \
 -keyout grafana-web.key \
 -out grafana-web.crt
```
### save Grafana web certificate to secret 
```
kubectl create secret tls grafana-web-tls \
  --namespace grafana-system \
  --key grafana-web.key \
  --cert grafana-web.crt
```
### create Grafana Ingress publication
```
cat <<EOT > grafana-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: grafana-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: "grafana.59ff44dd.nip.io"
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
            name: grafana-system
            port:
              number: 80
  tls:
    - hosts:
      - grafana.59ff44dd.nip.io
      secretName: grafana-web-tls
EOT
```
```
kubectl apply -f grafana-ingress.yaml
```
