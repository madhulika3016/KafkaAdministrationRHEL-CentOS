# Kafka KRaft Mode Setup — Step by Step

This document provides a clean step-by-step guide for setting up Apache Kafka in **KRaft mode** (without Zookeeper) on RHEL/CentOS.

---

## Step 0: Prerequisites

- Kafka downloaded and extracted to `/opt/kafka`
- Java installed (`java -version` should work)
- Clean environment (no old Kafka/Zookeeper running)

---

## Step 1: Create a fresh KRaft broker configuration

```bash
sudo tee /opt/kafka/config/server1.properties > /dev/null <<'EOF'
####################### Broker + Controller #######################
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@localhost:9093

####################### Listeners #######################
listeners=PLAINTEXT://localhost:9092,CONTROLLER://localhost:9093
controller.listener.names=CONTROLLER

####################### Log dirs #######################
log.dirs=/tmp/kraft-logs

####################### Topic Defaults ##################
num.partitions=1
default.replication.factor=1
min.insync.replicas=1

####################### Internal Topics ################
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1

####################### Socket / Network ################
socket.request.max.bytes=104857600
EOF
```

---

## Step 2: Format KRaft Storage

```bash
# Generate a unique cluster ID
UUID=$(uuidgen)

# Format storage
sudo /opt/kafka/bin/kafka-storage.sh format -t $UUID -c /opt/kafka/config/server1.properties
```

- This writes metadata to `/tmp/kraft-logs`
- Only required once per cluster

---

## Step 3: Start Kafka in KRaft Mode

```bash
# Start in foreground to watch logs
sudo /opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server1.properties

# After confirming it works, run in background
sudo /opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server1.properties
```

---

## Step 4: Create a Topic

```bash
sudo /opt/kafka/bin/kafka-topics.sh \
  --create \
  --topic test-topic \
  --partitions 3 \
  --replication-factor 1 \
  --bootstrap-server localhost:9092

# Verify topic exists
sudo /opt/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

---

## Step 5: Produce Messages

```bash
sudo /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic
```

Type messages and press Enter.

---

## Step 6: Consume Messages

```bash
sudo /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --from-beginning
```

You should see the messages you typed in the producer.

---

## Step 7: Optional — Clean Shutdown

```bash
sudo pkill -f kafka.Kafka
```

---

✅ This setup creates a single-node Kafka cluster in KRaft mode:

- No Zookeeper required
- Fresh logs in `/tmp/kraft-logs`
- Plaintext communication (no SSL/SASL)
- Single-node cluster ready for testing or development

