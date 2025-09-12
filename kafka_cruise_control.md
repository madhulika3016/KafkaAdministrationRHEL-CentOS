# 🚦 Kafka Cruise Control

## Overview
Cruise Control is an **open-source tool** (by LinkedIn, now part of Apache Kafka ecosystem) that helps you:  
- Monitor **cluster workload** (disk, CPU, network, partition sizes).  
- Detect **imbalances** across brokers.  
- Perform **partition reassignments automatically**.  
- Support **self-healing** after broker failures.  
- Offer **goal-based optimization** (rack-awareness, resource balance, etc.).  

---

## 1️⃣ Key Features
- **Automated Balancing**: Keeps partitions evenly spread across brokers.  
- **Goals-Based Optimization**: You define goals → Cruise Control finds the best plan.  
- **Anomaly Detection & Self-Healing**: Detects broker failures or offline replicas and triggers auto-fix.  
- **REST API**: Everything is exposed via REST (no extra CLI needed).  

---

## 2️⃣ Installing Cruise Control

### On Kafka Cluster
1. **Download Cruise Control**:
   ```bash
   git clone https://github.com/linkedin/cruise-control.git
   cd cruise-control
   ./gradlew jar
   ```

2. **Configure Cruise Control** (`config/cruisecontrol.properties`):
   ```properties
   bootstrap.servers=localhost:9092
   zookeeper.connect=localhost:2181   # (if using ZooKeeper mode)
   # For KRaft:
   metadata.max.age.ms=30000
   ```

   Set metrics reporters in Kafka broker configs (`server.properties`):
   ```properties
   metric.reporters=com.linkedin.kafka.cruisecontrol.metricsreporter.CruiseControlMetricsReporter
   cruise.control.metrics.reporter.bootstrap.servers=localhost:9092
   ```

3. **Start Cruise Control**:
   ```bash
   ./gradlew cruise-control-start
   ```

---

## 3️⃣ Using Cruise Control

### Check Cluster State
```bash
curl -X GET "http://localhost:9090/kafkacruisecontrol/state?substates=anomaly_detector,analyzer,executor"
```

### Get Optimization Proposal
```bash
curl -X GET "http://localhost:9090/kafkacruisecontrol/proposals"
```

### Execute Rebalance
```bash
curl -X POST "http://localhost:9090/kafkacruisecontrol/rebalance?dryRun=false"
```

### Fix Offline Replicas
```bash
curl -X POST "http://localhost:9090/kafkacruisecontrol/fix_offline_replicas"
```

### View Load
```bash
curl -X GET "http://localhost:9090/kafkacruisecontrol/load"
```

---

## 4️⃣ Goals Examples
Default goals include:
- `RackAwareGoal`  
- `ReplicaCapacityGoal`  
- `DiskCapacityGoal`  
- `NetworkInboundCapacityGoal`  
- `NetworkOutboundCapacityGoal`  
- `CpuCapacityGoal`  
- `TopicReplicaDistributionGoal`  

👉 You can prioritize which goals matter most.  

---

## 5️⃣ Self-Healing
Enable in `cruisecontrol.properties`:
```properties
self.healing.enabled=true
```

Cruise Control will auto-trigger fixes for:
- Dead brokers.  
- Offline replicas.  
- Goal violations.  

---

## 6️⃣ UI & Monitoring
- Cruise Control has a **web UI** (separate repo: `cruise-control-ui`).  
- Prometheus exporters available for monitoring rebalance progress & cluster health.  

---

✅ In practice: You run Cruise Control as a sidecar service with Kafka, point it to your cluster, and let it handle **rebalancing and broker load automatically**.
