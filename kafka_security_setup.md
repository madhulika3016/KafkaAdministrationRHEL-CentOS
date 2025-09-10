# Kafka Security Setup with SASL_SSL, ACLs, and SSL Certificates

This guide walks through securing Apache Kafka with SSL certificates, SASL/PLAIN authentication, and ACLs for producers and consumers.

---

## 1. Generate SSL Certificates

```bash
sudo mkdir -p /opt/kafka/certs
cd /opt/kafka/certs

# Generate broker keystore
sudo keytool -genkey -noprompt   -alias kafka-broker   -dname "CN=localhost, OU=Kafka, O=Company, L=City, S=State, C=IN"   -keystore kafka.server.keystore.jks   -keyalg RSA -storepass brokerpass -keypass brokerpass -validity 365

# Export broker certificate
sudo keytool -export   -alias kafka-broker   -file kafka-broker.cer   -keystore kafka.server.keystore.jks   -storepass brokerpass

# Create truststore
sudo keytool -import -noprompt   -alias kafka-broker   -file kafka-broker.cer   -keystore kafka.server.truststore.jks   -storepass brokerpass
```

---

## 2. Configure Kafka Broker for SASL_SSL

Edit `/opt/kafka/config/server.properties`:

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

## 3. Configure JAAS for Broker

Create `/opt/kafka/config/kafka_server_jaas.conf`:

```bash
sudo tee /opt/kafka/config/kafka_server_jaas.conf > /dev/null <<'EOF'
KafkaServer {
   org.apache.kafka.common.security.plain.PlainLoginModule required
   username="admin"
   password="admin-secret"
   user_admin="admin-secret"
   user_producer1="prod1-secret"
   user_consumer1="cons1-secret";
};
EOF
```

Start Kafka with JAAS:

```bash
sudo KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/kafka_server_jaas.conf" /opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties
```

---

## 4. Create JAAS Configs for Clients

### Producer JAAS Config

```bash
sudo tee /opt/kafka/config/producer1_jaas.conf > /dev/null <<'EOF'
KafkaClient {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="producer1"
  password="prod1-secret";
};
EOF
```

### Consumer JAAS Config

```bash
sudo tee /opt/kafka/config/consumer1_jaas.conf > /dev/null <<'EOF'
KafkaClient {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="consumer1"
  password="cons1-secret";
};
EOF
```

---

## 5. Add ACLs

### Allow producer1 to write:

```bash
sudo /opt/kafka/bin/kafka-acls.sh   --authorizer-properties zookeeper.connect=localhost:2181   --add --allow-principal User:producer1   --operation Write   --topic secure-orders
```

### Allow consumer1 to read (topic + group):

```bash
sudo /opt/kafka/bin/kafka-acls.sh   --authorizer-properties zookeeper.connect=localhost:2181   --add --allow-principal User:consumer1   --operation Read   --topic secure-orders

sudo /opt/kafka/bin/kafka-acls.sh   --authorizer-properties zookeeper.connect=localhost:2181   --add --allow-principal User:consumer1   --operation Read   --group secure-group
```

---

## 6. Client Config Properties

Create `/opt/kafka/config/client-config.properties`:

```bash
sudo tee /opt/kafka/config/client-config.properties > /dev/null <<'EOF'
security.protocol=SASL_SSL
sasl.mechanism=PLAIN

# Truststore for SSL
ssl.truststore.location=/opt/kafka/certs/kafka.server.truststore.jks
ssl.truststore.password=brokerpass
EOF
```

---

## 7. Create Secure Topic

```bash
sudo KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/producer1_jaas.conf" /opt/kafka/bin/kafka-topics.sh   --create   --topic secure-orders   --partitions 3   --replication-factor 1   --bootstrap-server localhost:9093   --command-config /opt/kafka/config/client-config.properties
```

---

## 8. Use with Producer

```bash
sudo KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/producer1_jaas.conf" /opt/kafka/bin/kafka-console-producer.sh   --bootstrap-server localhost:9093   --topic secure-orders   --producer.config /opt/kafka/config/client-config.properties
```

---

## 9. Use with Consumer

```bash
sudo KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/consumer1_jaas.conf" /opt/kafka/bin/kafka-console-consumer.sh   --bootstrap-server localhost:9093   --topic secure-orders   --group secure-group   --consumer.config /opt/kafka/config/client-config.properties   --from-beginning
```

---

✅ You now have a fully working **Kafka cluster secured with SSL, SASL/PLAIN, and ACLs**.
