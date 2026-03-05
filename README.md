3️⃣ Install Node Exporter on Jenkins EC2 -> to get the logs of in prometheus which is running in k8s cluster, jenkins which is running on ec2 instance.
Run these commands on the Jenkins EC2 server:

cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar xvf node_exporter-1.8.1.linux-amd64.tar.gz
cd node_exporter-1.8.1.linux-amd64
sudo cp node_exporter /usr/local/bin/

2. Run Node Exporter
Start it:
 -->    node_exporter

Now test locally:
-->    curl localhost:9100/metrics

http://PROMETHEUS-IP:9090/targets

Run below query on prometheus and next fectch from grafana by giving data source connection
rate(node_cpu_seconds_total{instance="3.110.180.58:9100",mode!="idle"}[5m])


-------******-------
Steps to send logs to loki
Correct Way to Send Jenkins Logs to Loki
You already did most of it correctly with:
/etc/promtail-config.yaml

But Jenkins installed via systemd usually logs to journald, not /var/log/jenkins/jenkins.log.

Your command confirmed that:
~  journalctl -u jenkins

So Promtail must read systemd logs, not file logs.
Correct Promtail Config for Jenkins (FINAL):
On the Jenkins EC2 server use this:
-------------------------------------------------------
~  sudo vi /etc/promtail-config.yaml
server:
  http_listen_port: 9081
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml
  
clients:
  - url: http://65.0.67.196:32000/loki/api/v1/push

scrape_configs:
  - job_name: jenkins-journal
    journal:
      path: /var/log/journal
      labels:
        job: jenkins
        instance: jenkins-ec2
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        regex: 'jenkins.service'
        action: keep
-----------------------------------------------------------
Replace:  65.0.67.196 --> with your Loki NodePort IP.

Restart Promtail
  Stop old one:
~  sudo pkill promtail

  Start again:
~  sudo promtail -config.file=/etc/promtail-config.yaml

Verify Logs Are Being Sent
You should see something like:
msg="tailing new journal target"

sudo chown -R 472:472 /mnt/data/grafana
sudo chmod -R 775 /mnt/data/grafana ->when we create pv and pvc should take care of permissions of volumes and monunt them

Query in Grafana Loki
Use this below query:
~  {job="jenkins"}
   or
~  {instance="jenkins-ec2"}
