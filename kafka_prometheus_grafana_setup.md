# Kafka 3.8 Monitoring with Prometheus and Grafana on RHEL 10 ARM64 / Mac M1

## 1️⃣ Install prerequisites
```bash
sudo dnf update -y
sudo dnf install java-17-openjdk wget tar git -y
```

## 2️⃣ Install Kafka 3.8
```bash
cd /opt
sudo wget https://downloads.apache.org/kafka/3.8.0/kafka_2.13-3.8.0.tgz
sudo tar -xvzf kafka_2.13-3.8.0.tgz
sudo mv kafka_2.13-3.8.0 kafka
```

## 3️⃣ Set up Kafka JMX Exporter
```bash
cd /opt/kafka
sudo mkdir -p jmx_exporter
cd jmx_exporter

# Download JMX Prometheus agent
sudo wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar

# Create minimal Kafka JMX config
sudo tee kafka-jmx.yml > /dev/null <<EOF
rules:
  - pattern: "kafka.server<type=(.+), name=(.+)><>Value"
    name: "kafka_server_\$1_\$2"
    type: GAUGE
EOF
```

## 4️⃣ Configure Kafka to use JMX Exporter
```bash
sudo tee /opt/kafka/kafka_env.sh > /dev/null <<EOF
export KAFKA_OPTS="-javaagent:/opt/kafka/jmx_exporter/jmx_prometheus_javaagent-0.20.0.jar=7071:/opt/kafka/jmx_exporter/kafka-jmx.yml"
EOF

sudo chmod +x /opt/kafka/kafka_env.sh
```
Source the environment before starting Kafka:
```bash
source /opt/kafka/kafka_env.sh
```

## 5️⃣ Start Kafka (Zookeeper first)
```bash
cd /opt/kafka

# Start Zookeeper
sudo bin/zookeeper-server-start.sh config/zookeeper.properties &

# Start Kafka
sudo bin/kafka-server-start.sh config/server.properties &
```

## 6️⃣ Install Prometheus (ARM64) on port 9095
```bash
cd /opt
sudo wget https://github.com/prometheus/prometheus/releases/download/v2.49.0/prometheus-2.49.0.linux-arm64.tar.gz
sudo tar -xvzf prometheus-2.49.0.linux-arm64.tar.gz
sudo mv prometheus-2.49.0.linux-arm64 prometheus
cd prometheus
```

## 7️⃣ Create Prometheus data directories and fix permissions
```bash
sudo mkdir -p /opt/prometheus/data /opt/prometheus/wal
sudo chown -R $(whoami):$(whoami) /opt/prometheus/data /opt/prometheus/wal
sudo chmod 700 /opt/prometheus/data /opt/prometheus/wal
```

## 8️⃣ Create Prometheus configuration
```bash
sudo tee /opt/prometheus/prometheus.yml > /dev/null <<EOF
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'kafka_jmx'
    static_configs:
      - targets: ['localhost:7071']
    metrics_path: '/metrics'
    scheme: 'http'

  - job_name: 'kafka_exporter'
    static_configs:
      - targets: ['localhost:9308']
    metrics_path: '/metrics'
    scheme: 'http'
EOF
```

## 9️⃣ Run Prometheus
```bash
cd /opt/prometheus
./prometheus \
  --config.file=prometheus.yml \
  --web.listen-address=":9095" \
  --storage.tsdb.path=/opt/prometheus/data \
  --storage.tsdb.wal-directory=/opt/prometheus/wal
```
Prometheus UI: [http://localhost:9095](http://localhost:9095)

## 10️⃣ Install Grafana
```bash
sudo dnf install grafana -y
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```
Grafana UI: [http://localhost:3000](http://localhost:3000)

- Default login: `admin/admin`
- Add Prometheus as data source: `http://localhost:9095`
- Import Kafka dashboards: IDs `721` or `7587`

