Configure cluster monitoring


Deploy Prometheus
```
sudo bash
mkdir /opt/prometheus
cd /opt/prometheus
curl https://raw.githubusercontent.com/prometheus/prometheus/main/documentation/examples/prometheus.yml > /opt/prometheus/prometheus.yml
exit


sudo docker run -d -p 9090:9090 -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml --restart=always prom/prometheus
```

Add prometheus targets
```
echo "
  - job_name: 'nodes'
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.0.201:9100','192.168.0.202:9100','192.168.0.203:9100','192.168.0.204:9100','192.168.0.205:9100','192.168.0.206:9100','192.168.0.207:9100']" >> /opt/prometheus/prometheus.yml
```

Deploy node exporter
```
https://back2basics.io/2020/04/setup-a-prometheus-node-exporter-on-ubuntu-18-04-server-for-the-raspberry-pi/
```

Deploy Grafana

run initial container
```
sudo docker run grafana/grafana:6.5.0 
```

copy /etc/grafana to /opt to node
```
sudo docker cp <containerID>:/var/lib/grafana /opt
sudo docker stop <containerID>
sudo chown 472:472 --recursive /opt/grafana
```

start container
```
sudo docker run -d -p 3000:3000 --name grafana -v /opt/grafana:/var/lib/grafana --restart=always grafana/grafana:6.5.0 
```
default grafana login: admin/admin


check stress test
```
sudo apt install stress
stress --cpu 32 --io 4 --vm 2 --vm-bytes 128M
```