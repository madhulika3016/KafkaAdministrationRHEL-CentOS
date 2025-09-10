# Kafka Schema Registry Setup for SASL_SSL Cluster

This guide sets up Schema Registry with SASL_SSL secured Kafka, including JAAS, SSL, ACLs, and the `_schemas` topic.

---

## Step 1: Create JAAS file for Schema Registry

```bash
sudo mkdir -p /opt/schema-registry/etc

sudo tee /opt/schema-registry/etc/schema-registry_jaas.conf > /dev/null <<'EOF'
KafkaClient {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="admin"
  password="admin-secret";
};
EOF
```

---

## Step 2: Create client-config.properties

```bash
sudo tee /opt/schema-registry/etc/client-config.properties > /dev/null <<'EOF'
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
ssl.truststore.location=/opt/kafka/certs/kafka.server.truststore.jks
ssl.truststore.password=brokerpass
EOF
```

---

## Step 3: Configure Schema Registry properties

```bash
sudo tee /opt/schema-registry/etc/schema-registry.properties > /dev/null <<'EOF'
# Schema Registry listener
listeners=https://0.0.0.0:8081

# Kafka cluster connection
kafkastore.bootstrap.servers=SASL_SSL://localhost:9093
kafkastore.security.protocol=SASL_SSL
kafkastore.sasl.mechanism=PLAIN
kafkastore.ssl.truststore.location=/opt/kafka/certs/kafka.server.truststore.jks
kafkastore.ssl.truststore.password=brokerpass

# Topic to store schemas
kafkastore.topic=_schemas

# Optional configs
debug=true
EOF
```

---

## Step 4: Create `_schemas` topic manually

```bash
sudo KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka/config/kafka_server_jaas.conf" \
/opt/kafka/bin/kafka-topics.sh \
  --create \
  --topic _schemas \
  --partitions 1 \
  --replication-factor 1 \
  --bootstrap-server localhost:9093 \
  --command-config /opt/schema-registry/etc/client-config.properties
```

---

## Step 5: Set ACLs for Schema Registry user

```bash
sudo /opt/kafka/bin/kafka-acls.sh \
  --authorizer-properties zookeeper.connect=localhost:2181 \
  --add --allow-principal User:admin --operation All --topic _schemas
```

---

## Step 6: Start Schema Registry with JAAS

```bash
sudo pkill -f schema-registry  # stop any previous instances

sudo KAFKA_OPTS="-Djava.security.auth.login.config=/opt/schema-registry/etc/schema-registry_jaas.conf" \
/opt/schema-registry/bin/schema-registry-start /opt/schema-registry/etc/schema-registry.properties
```

> Run without `-daemon` initially to watch logs for errors.
> After confirming it works, you can run with `-daemon`.

---

## Step 7: Test Schema Registry

```bash
curl --insecure https://localhost:8081/subjects
```

- Should return: `[]` (empty list of schemas).

---

## Step 8: Using Avro Producer/Consumer

Set the Schema Registry URL in your clients:

```properties
schema.registry.url=https://localhost:8081
basic.auth.credentials.source=USER_INFO
schema.registry.basic.auth.user.info=admin:admin-secret
```

This ensures your Avro producers and consumers can securely communicate with Schema Registry.

