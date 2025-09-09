# Kafka Full Setup Guide with Zookeeper, Multi-Broker, Postgres Connector, and Monitoring

## Table of Contents
1. [Create Kafka Topic](#create-kafka-topic)
2. [Start Producers and Consumers](#start-producers-and-consumers)
3. [Configure Multiple Brokers](#configure-multiple-brokers)
4. [Kafka Cluster Verification](#kafka-cluster-verification)
5. [Kafka Connect with PostgreSQL](#kafka-connect-with-postgresql)
6. [Kafka Monitoring with Prometheus and Grafana](#kafka-monitoring-with-prometheus-and-grafana)
7. [Uninstall Kafka](#uninstall-kafka)

---

## 1. Create Kafka Topic
```bash
/opt/kafka/bin/kafka-topics.sh --create --topic test --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
/opt/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

## 2. Start Producers and Consumers
### Producers
**Producer 1:**
```bash
/opt/kafka/bin/kafka-console-producer.sh --topic test --bootstrap-server localhost:9092
```
**Producer 2:**
```bash
/opt/kafka/bin/kafka-console-producer.sh --topic test --bootstrap-server localhost:9092
```

### Consumers
**Option A: Same Consumer Group (messages load-balanced)**
```bash
/opt/kafka/bin/kafka-console-consumer.sh --topic test --bootstrap-server localhost:9092 --group test-group
```
**Option B: Different Consumer Groups (both get all messages)**
```bash
/opt/kafka/bin/kafka-console-consumer.sh --topic test --bootstrap-server localhost:9092 --group group1
/opt/kafka/bin/kafka-console-consumer.sh --topic test --bootstrap-server localhost:9092 --group group2
```

---

## 3. Configure Multiple Brokers
Create 3 copies of `server.properties`:

**server-1.properties**
```properties
broker.id=1
listeners=PLAINTEXT://:9092
log.dirs=/tmp/kafka-logs-1
zookeeper.connect=localhost:2181
```

**server-2.properties**
```properties
broker.id=2
listeners=PLAINTEXT://:9093
log.dirs=/tmp/kafka-logs-2
zookeeper.connect=localhost:2181
```

**server-3.properties**
```properties
broker.id=3
listeners=PLAINTEXT://:9094
log.dirs=/tmp/kafka-logs-3
zookeeper.connect=localhost:2181
```

### Start Brokers
```bash
bin/kafka-server-start.sh config/server-1.properties &
bin/kafka-server-start.sh config/server-2.properties &
bin/kafka-server-start.sh config/server-3.properties &
```

### Verify Cluster
```bash
bin/zookeeper-shell.sh localhost:2181 ls /brokers/ids
# Should show [1,2,3]
```

### Create Replicated Topic
```bash
bin/kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --partitions 3 --replication-factor 3
bin/kafka-topics.sh --describe --topic test-topic --bootstrap-server localhost:9092
```

---

## 4. Kafka Cluster Behavior
- Leader handles reads/writes
- Replicas store copies across brokers
- ISR (In-Sync Replicas) are healthy copies
- High availability, scalability, and fault tolerance ensured

### Test Producer/Consumer
```bash
bin/kafka-console-producer.sh --topic test-topic --bootstrap-server localhost:9092
bin/kafka-console-consumer.sh --topic test-topic --from-beginning --bootstrap-server localhost:9092
```

### Stop and Restart Broker
```bash
sudo pkill -f "kafka.Kafka config/server-1.properties"
bin/kafka-server-start.sh config/server-1.properties &
```

---

## 5. Kafka Connect with PostgreSQL
### Install PostgreSQL
```bash
sudo dnf update -y
sudo dnf install -y postgresql-server postgresql-contrib
sudo postgresql-setup --initdb
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

### Configure Auth
Edit `/var/lib/pgsql/data/pg_hba.conf` to use `md5` for local and host connections, then restart:
```bash
sudo systemctl restart postgresql
```

### Create Database, User, and Table
```sql
sudo -u postgres psql
CREATE USER kafkauser WITH PASSWORD 'kafkapass';
CREATE DATABASE kafkadb OWNER kafkauser;
\c kafkadb
CREATE TABLE customer (id SERIAL PRIMARY KEY, firstname TEXT, lastname TEXT, email TEXT);
INSERT INTO customer (firstname, lastname, email) VALUES ('Alice','Wonder','alice@example.com'),('Bob','Marley','bob@example.com');
GRANT CONNECT ON DATABASE kafkadb TO kafkauser;
GRANT USAGE ON SCHEMA public TO kafkauser;
GRANT SELECT ON customer TO kafkauser;
\q
```

Test connection:
```bash
PGPASSWORD=kafkapass psql -U kafkauser -d kafkadb -h localhost -c "SELECT * FROM customer;"
```

### Kafka Connect Setup
1. Download & install Kafka
2. Start Zookeeper & Kafka
3. Create topic for Postgres table
4. Install JDBC driver & connector
5. Start Kafka Connect worker
6. Create JDBC source connector config
7. Consume data from Kafka
8. Create sink table and JDBC sink connector
9. Verify data in PostgreSQL

---

## 6. Kafka Monitoring with Prometheus and Grafana
### JMX Exporter
1. Download JMX exporter jar
2. Create `kafka-jmx.yml` configuration
3. Start Kafka with JMX agent

### Prometheus
1. Download & extract Prometheus
2. Create `prometheus.yml` config
3. Start Prometheus
4. Verify via `http://localhost:9090` (or custom port)

### Grafana
1. Download & extract Grafana
2. Start Grafana server
3. Access `http://localhost:3000` (admin/admin)
4. Add Prometheus as data source
5. Import Kafka dashboards (IDs 7587, 10466)

### Optional: Postgres Exporter
1. Install postgres exporter
2. Configure monitoring user
3. Set `DATA_SOURCE_NAME`
4. Run exporter on port 9187
5. Add scrape job in Prometheus
6. Import Grafana dashboard ID 9628

---

## 7. Uninstall Kafka
```bash
sudo systemctl stop kafka
sudo systemctl stop zookeeper
sudo systemctl disable kafka
sudo systemctl disable zookeeper
sudo pkill -f kafka
sudo pkill -f zookeeper
sudo dnf remove -y kafka
sudo rm -rf /opt/kafka /var/lib/kafka /var/log/kafka /etc/kafka /tmp/kafka-logs /tmp/zookeeper
which kafka-server-start.sh
sudo systemctl status kafka
```

This concludes the full Kafka setup guide including Zookeeper, multi-broker replication, Kafka Connect with PostgreSQL, Prometheus + Grafana monitoring, and uninstall steps.

