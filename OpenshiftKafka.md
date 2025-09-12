
# OpenShift (oc) CLI and Kafka on OpenShift - Cheat Sheet

## OpenShift CLI (oc) Basics

```bash
# Login to OpenShift cluster
oc login https://<openshift_api_url>:6443 -u <username> -p <password>

# Check current project/namespace
oc project

# List all projects
oc get projects

# Switch project
oc project <project_name>

# List pods, services, deployments
oc get pods
oc get svc
oc get deploy

# Describe a resource
oc describe pod <pod_name>

# Get pod logs
oc logs <pod_name>

# Execute command inside a pod
oc exec -it <pod_name> -- /bin/bash
```

## Deploy Kafka on OpenShift

```bash
# Create a new project for Kafka
oc new-project kafka-demo

# Deploy Kafka using Strimzi Operator
oc apply -f 'https://strimzi.io/install/latest?namespace=kafka-demo' -n kafka-demo

# Create a Kafka cluster
oc apply -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka-demo
spec:
  kafka:
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
    storage:
      type: ephemeral
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF
```

## Kafka Commands on OpenShift

```bash
# List topics
oc exec -it my-cluster-kafka-0 -c kafka -- bin/kafka-topics.sh --list --bootstrap-server localhost:9092

# Create a topic
oc exec -it my-cluster-kafka-0 -c kafka -- bin/kafka-topics.sh --create --topic my-topic --partitions 3 --replication-factor 3 --bootstrap-server localhost:9092

# Produce messages
oc exec -it my-cluster-kafka-0 -c kafka -- bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-topic

# Consume messages
oc exec -it my-cluster-kafka-0 -c kafka -- bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-topic --from-beginning
```

## Kafka MirrorMaker 2 on OpenShift

```bash
# Deploy MirrorMaker 2
oc apply -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: my-mm2
  namespace: kafka-demo
spec:
  version: 3.8.0
  replicas: 1
  connectCluster: target
  clusters:
    - alias: source
      bootstrapServers: source-kafka-bootstrap:9092
    - alias: target
      bootstrapServers: target-kafka-bootstrap:9092
  mirrors:
    - sourceCluster: source
      targetCluster: target
      sourceConnector:
        config:
          replication.factor: 1
      topicsPattern: ".*"
EOF
```

---

