# Kafka MirrorMaker 2 Setup and Backup Methods

## MirrorMaker 2 with Two Zookeeper Clusters – Step by Step

**Assumptions:**
- Kafka installed at `/opt/kafka`
- Cluster A (source): Zookeeper 2181, brokers 9092, 9093
- Cluster B (target): Zookeeper 2182, brokers 9094, 9095
- No security (plain PLAINTEXT)

---

### Step 1: Start Zookeeper Servers

**Cluster A:**
```bash
sudo /opt/kafka/bin/zookeeper-server-start.sh -daemon /opt/kafka/config/zookeeperA.properties
```
**Sample `zookeeperA.properties`**
```
tickTime=2000
dataDir=/tmp/zookeeperA
clientPort=2181
initLimit=5
syncLimit=2
```

**Cluster B:**
```bash
sudo /opt/kafka/bin/zookeeper-server-start.sh -daemon /opt/kafka/config/zookeeperB.properties
```
**Sample `zookeeperB.properties`**
```
tickTime=2000
dataDir=/tmp/zookeeperB
clientPort=2182
initLimit=5
syncLimit=2
```

---

### Step 2: Start Kafka Brokers

**Cluster A Broker 1 (9092):**
```bash
sudo /opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/serverA1.properties
```
**`serverA1.properties`:**
```
broker.id=1
listeners=PLAINTEXT://:9092
log.dirs=/tmp/kafka-logs-A1
zookeeper.connect=localhost:2181
num.partitions=3
offsets.topic.replication.factor=1
```

**Cluster A Broker 2 (9093):**
```bash
sudo /opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/serverA2.properties
```
**`serverA2.properties`:**
```
broker.id=2
listeners=PLAINTEXT://:9093
log.dirs=/tmp/kafka-logs-A2
zookeeper.connect=localhost:2181
num.partitions=3
offsets.topic.replication.factor=1
```

**Cluster B Broker 1 (9094):**
```bash
sudo /opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/serverB1.properties
```
**`serverB1.properties`:**
```
broker.id=1
listeners=PLAINTEXT://:9094
log.dirs=/tmp/kafka-logs-B1
zookeeper.connect=localhost:2182
num.partitions=3
offsets.topic.replication.factor=1
```

**Cluster B Broker 2 (9095):**
```bash
sudo /opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/serverB2.properties
```
**`serverB2.properties`:**
```
broker.id=2
listeners=PLAINTEXT://:9095
log.dirs=/tmp/kafka-logs-B2
zookeeper.connect=localhost:2182
num.partitions=3
offsets.topic.replication.factor=1
```

---

### Step 3: Create Sample Topics in Source Cluster A
```bash
sudo /opt/kafka/bin/kafka-topics.sh --create --topic test-topic1 --partitions 3 --replication-factor 2 --zookeeper localhost:2181
sudo /opt/kafka/bin/kafka-topics.sh --create --topic test-topic2 --partitions 2 --replication-factor 2 --zookeeper localhost:2181
```

---

### Step 4: Configure MirrorMaker 2

```bash
sudo mkdir -p /opt/kafka/mm2
cd /opt/kafka/mm2
```

**`mm2.properties`:**
```properties
# Define clusters
clusters = A, B

# Cluster A (source)
A.bootstrap.servers=localhost:9092
A.zookeeper.connect=localhost:2181

# Cluster B (target)
B.bootstrap.servers=localhost:9094
B.zookeeper.connect=localhost:2182

# Replication configuration
A->B.enabled=true
A->B.topics=.*         # replicate all topics
A->B.groups=.*         # replicate all consumer groups

# Replication policy
replication.policy.class=org.apache.kafka.connect.mirror.DefaultReplicationPolicy
offset.syncs.topic.replication.factor=1
```

---

### Step 5: Start MirrorMaker 2

**Foreground (for logs):**
```bash
/opt/kafka/bin/connect-mirror-maker.sh /opt/kafka/mm2/mm2.properties
```

**Background:**
```bash
nohup /opt/kafka/bin/connect-mirror-maker.sh /opt/kafka/mm2/mm2.properties > mm2.log 2>&1 &
```

---

### Step 6: Verify Replication

**List topics in Cluster B:**
```bash
/opt/kafka/bin/kafka-topics.sh --list --zookeeper localhost:2182
```

**Check consumer groups in Cluster B:**
```bash
/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9094 --list
```

**Test producing/consuming:**
- Produce to Cluster A:
```bash
/opt/kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test-topic1
```
- Consume from Cluster B:
```bash
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9094 --topic test-topic1 --from-beginning
```

---

## Other Kafka Backup Methods

### 1. MirrorMaker 2 (Cross-Cluster Replication)
- Real-time replication of topics & consumer groups.
- Requires a second cluster.
- Example `mm2.properties`:
```properties
clusters = source, backup
source.bootstrap.servers=localhost:9092
backup.bootstrap.servers=localhost:9097
source->backup.enabled=true
source->backup.topics=.*
```
- Start: `./bin/connect-mirror-maker.sh mm2.properties`

### 2. kafka-console-consumer + kafka-console-producer (Dump & Restore)
**Backup:**
```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-topic --from-beginning --timeout-ms 60000 > my-topic-backup.json
```
**Restore:**
```bash
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic my-topic < my-topic-backup.json
```
- Pros: Simple, no extra cluster.
- Cons: Small topics only; may lose metadata.

### 3. kafka-dump-log Tools
```bash
kafka-dump-log.sh --files /kafka-logs/my-topic-0/00000000000000000000.log --print-data-log
```
- Pros: Broker-level, low-level backup.
- Cons: Not for very large clusters.

### 4. Tiered Storage / Confluent Cloud Snapshots
- Kafka-native snapshots to cloud (S3). Managed & automatic.
- Cons: Proprietary; requires Confluent.

### 5. Filesystem Snapshots
- Stop broker, copy `log.dirs` to backup, restart broker.
- Pros: Exact replica.
- Cons: Broker downtime required.

---

**Recommendation:**
- For production: **MirrorMaker 2** (real-time, reliable).  
- For small topics or ad-hoc: **console consumer/producer dump**.  
- For full broker backup: **filesystem snapshot with proper shutdown**.

