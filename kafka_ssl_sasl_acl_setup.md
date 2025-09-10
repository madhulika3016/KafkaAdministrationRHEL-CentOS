# Kafka Secure Setup with SSL/SASL/ACLs

This guide shows step-by-step how to configure **Kafka broker with SSL and SASL/PLAIN security**, create **producer/consumer JAAS configs**, create a topic, set ACLs, and test message flow.

---

## 1️⃣ Generate SSL Certificates

Create directory for certificates:
```bash
sudo mkdir -p /opt/kafka/certs
cd /opt/kafka/certs
```

Generate self-signed certificate for broker:
```bash
sudo keytool -genkey -noprompt \
  -alias kafka-broker \
  -dname "CN=localhost, OU=Kafka, O=Company, L=City, S=State, C=IN" \
  -keystore kafka.server.keystore.jks \
  -keyalg RSA -storepass brokerpass -keypass brokerpass -validity 365
```

Export certificate:
```bash
sudo keytool -export \
  -alias kafka-broker \
  -file kafka-broker.cer \
  -keystore kafka.server.keystore.jks \
  -storepass brokerpass
```

Create truststore for clients:
```bash
sudo keytool -import -noprompt \
  -alias kafka-broker \
  -file kafka-broker.cer \
  -keystore kafka.server.truststore.jks \
  -storepass brokerpass
```

---

## 2️⃣ Configure Kafka Broker for SASL_SSL

Edit `server.properties`:
```bash
sudo tee -a /opt/kafka/config/server.properties > /dev/null <<'EOF'

listeners=SASL_SSL://:9093
advertised.listeners=SASL_SSL://localhost:9093
listener.security.protocol.map=SASL_SSL:SASL_SSL
ssl.keystore.location=/opt/kafka/certs/kafka.server.keystore.jks
ssl.keystore.password=brokerpass
ssl.key.password=brokerpass
ssl.truststore.location=/opt/kafka/certs/kafka.server.truststore.jks
ssl.truststore.password=brokerpass
sasl.mechanism.inter.broker.protocol=PLAIN
security.inter.broker.protocol=SASL_SSL
sasl.enabled.mechanisms=PLAIN
EOF
```

---

## 3️⃣ Configure JAAS for Broker

Create `kafka_server_jaas.conf`:
```bash
sudo tee /opt/kafka/config/kafka_server_jaas.conf > /dev/null <<'EOF'
KafkaServer {
   org.apache.kafka.common.security.plain.PlainLoginModule required
   username="admin"
   password="admin-secret"
   user_admin="admin-secret"
   user_kafkauser="kafkapass";
};
EOF
```

Start Kafka with JAAS:
```bash
sudo KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/kafka_server_jaas.conf" \
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties
```

---

## 4️⃣ Create Producer JAAS Config
```bash
sudo tee /opt/kafka/config/producer_jaas.conf > /dev/null <<'EOF'
KafkaClient {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="producer1"
  password="prod1-secret";
};
EOF
```

## 5️⃣ Create Consumer JAAS Config
```bash
sudo tee /opt/kafka/config/consumer_jaas.conf > /dev/null <<'EOF'
KafkaClient {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="consumer1"
  password="cons1-secret";
};
EOF
```

---

## 6️⃣ Create a Topic
```bash
sudo /opt/kafka/bin/kafka-topics.sh \
  --create \
  --topic secure-orders \
  --partitions 1 \
  --replication-factor 1 \
  --bootstrap-server localhost:9092
```

---

## 7️⃣ Add ACLs

Allow producer1 to write:
```bash
sudo /opt/kafka/bin/kafka-acls.sh \
  --authorizer-properties zookeeper.connect=localhost:2181 \
  --add --allow-principal User:producer1 --operation Write --topic secure-orders
```

Allow consumer1 to read topic & group:
```bash
sudo /opt/kafka/bin/kafka-acls.sh \
  --authorizer-properties zookeeper.connect=localhost:2181 \
  --add --allow-principal User:consumer1 --operation Read --topic secure-orders

sudo /opt/kafka/bin/kafka-acls.sh \
  --authorizer-properties zookeeper.connect=localhost:2181 \
  --add --allow-principal User:consumer1 --operation Read --group secure-group
```

---

## 8️⃣ Start Producer
```bash
sudo KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/producer_jaas.conf" \
/opt/kafka/bin/kafka-console-producer.sh \
  --broker-list localhost:9092 \
  --topic secure-orders
```
Type messages and hit Enter.

---

## 9️⃣ Start Consumer
```bash
sudo KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/consumer_jaas.conf" \
/opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic secure-orders \
  --group secure-group \
  --from-beginning
```

You should now see all producer messages appearing in the consumer. ✅

