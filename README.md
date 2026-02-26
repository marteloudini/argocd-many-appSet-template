# Kubernetes Cluster Deployment with ArgoCD, Camel-K, Strimzi Kafka, and Database Connectors

This project contains YAML deployment scripts for a complete Kubernetes cluster setup with:

1. **Kubernetes Cluster (K8s v1.35)** - 3 control planes + 3 workers
2. **ArgoCD** - GitOps continuous delivery platform (admin/admin)
3. **Apache Camel-K** - Integration platform operator (all namespaces)
4. **Apache Kafka with Strimzi** - KRaft mode, 3 nodes (controller + broker)
5. **Kafka Connect Connectors** - Oracle, Salesforce, and PostgreSQL source/sink connectors

## Prerequisites

- Kubernetes v1.35+ cluster (or compatible version)
- kubectl configured with cluster access
- Minimum resources: 8 CPU cores, 32GB RAM
- Persistent storage for Kafka (50GB+ per node)

## Directory Structure

```
├── cluster/                      # Kubernetes cluster setup scripts
│   ├── kubeadm-config.yaml       # Kubeadm configuration
│   ├── cluster-vars.yml          # Cluster variables
│   ├── master-setup.sh           # First control plane setup
│   ├── control-plane-join.sh     # Join additional control planes
│   └── worker-join.sh            # Join worker nodes
│
├── argocd/                       # ArgoCD installation
│   ├── 00-namespace.yaml         # ArgoCD namespace
│   ├── 01-argocd-install.yaml    # ArgoCD components
│   ├── 02-argocd-admin-password.sh # Admin password setup
│   └── install-argocd.sh         # Installation script
│
├── camel-k/                      # Apache Camel-K operator
│   ├── 00-namespace.yaml         # Camel-K namespace
│   ├── 01-camel-k-operator.yaml  # Operator installation
│   └── install-camel-k.sh        # Installation script
│
├── strimzi/                      # Strimzi Kafka operator
│   ├── 00-namespace.yaml         # Kafka namespace
│   ├── 01-strimzi-operator.yaml  # Operator installation
│   ├── 02-kafka-cluster.yaml     # Kafka cluster (KRaft mode)
│   ├── 03-storage-class.yaml     # Storage configuration
│   └── install-strimzi.sh        # Installation script
│
├── kafka-connect/                # Kafka Connect connectors
│   ├── 00-kafka-connect.yaml     # Kafka Connect cluster
│   ├── 01-oracle-source-connector.yaml   # Oracle source
│   ├── 02-oracle-sink-connector.yaml     # Oracle sink
│   ├── 03-salesforce-connectors.yaml     # Salesforce source/sink
│   ├── 04-postgresql-connectors.yaml     # PostgreSQL source/sink
│   ├── oracle-prerequisites.sql          # Oracle DB setup
│   └── install-connectors.sh    # Installation script
```

## Installation Order

Follow these steps in order:

### 1. Kubernetes Cluster Setup

```bash
# On first control plane node
cd cluster
chmod +x master-setup.sh
./master-setup.sh

# Save the join command output for later use
```

### 2. Join Additional Nodes

```bash
# On second/third control plane node
cd cluster
chmod +x control-plane-join.sh
./control-plane-join.sh 2 <MASTER_IP> <JOIN_TOKEN> <CA_CERT_HASH>
./control-plane-join.sh 3 <MASTER_IP> <JOIN_TOKEN> <CA_CERT_HASH>

# On worker nodes
cd cluster
chmod +x worker-join.sh
./worker-join.sh 1 <MASTER_IP> <JOIN_TOKEN> <CA_CERT_HASH>
./worker-join.sh 2 <MASTER_IP> <JOIN_TOKEN> <CA_CERT_HASH>
./worker-join.sh 3 <MASTER_IP> <JOIN_TOKEN> <CA_CERT_HASH>
```

### 3. Install ArgoCD

```bash
cd argocd
chmod +x install-argocd.sh
./install-argocd.sh
```

**Admin Credentials:**

- Username: `admin`
- Password: `admin`

Access: `kubectl port-forward svc/argocd-server -n argocd 8080:80`

### 4. Install Camel-K

```bash
cd camel-k
chmod +x install-camel-k.sh
./install-camel-k.sh
```

### 5. Install Strimzi Kafka

```bash
cd strimzi
chmod +x install-strimzi.sh
./install-strimzi.sh
```

**Kafka Access:**

- Internal: `my-cluster-kafka-bootstrap.kafka:9092`
- External: Node ports 32100, 32101, 32102

### 6. Install Kafka Connect Connectors

```bash
cd kafka-connect
chmod +x install-connectors.sh
./install-connectors.sh
```

## Kafka Connect Connectors

### Oracle Connectors

**Source Connector** ([`01-oracle-source-connector.yaml`](kafka-connect/01-oracle-source-connector.yaml)):

- Uses Debezium Oracle Connector with LogMiner
- Captures CDC from Oracle database
- Topics: `oracle-source-topic`

**Sink Connector** ([`02-oracle-sink-connector.yaml`](kafka-connect/02-oracle-sink-connector.yaml)):

- Writes Kafka data to Oracle
- Supports upsert operations
- Topics: `oracle-sink-topic`

### Salesforce Connectors

**Source Connector** ([`03-salesforce-connectors.yaml`](kafka-connect/03-salesforce-connectors.yaml)):

- Uses Debezium Salesforce Connector
- Tracks Salesforce objects (Account, Contact, Lead, Opportunity, etc.)
- Topics: `salesforce.Account`, `salesforce.Contact`, etc.

**Sink Connector** ([`03-salesforce-connectors.yaml`](kafka-connect/03-salesforce-connectors.yaml)):

- Writes data back to Salesforce
- Supports SObject operations
- Topics: `salesforce-Account`, `salesforce-Contact`, etc.

### PostgreSQL Connectors

**Source Connector** ([`04-postgresql-connectors.yaml`](kafka-connect/04-postgresql-connectors.yaml)):

- Uses Debezium PostgreSQL Connector
- Captures CDC from PostgreSQL tables
- Topics: `postgres.users`, `postgres.orders`, `postgres.products`, `postgres.customers`

**Sink Connector** ([`04-postgresql-connectors.yaml`](kafka-connect/04-postgresql-connectors.yaml)):

- Writes Kafka data to PostgreSQL
- Supports upsert and delete operations
- Topics: `postgres.users`, `postgres.orders`, etc.

## Database Prerequisites

### Oracle Database

See [`kafka-connect/oracle-prerequisites.sql`](kafka-connect/oracle-prerequisites.sql) for:

- Enable archivelog mode
- Enable supplemental logging
- Create XStream user
- Create Kafka Connect user

### Salesforce

1. Create a Connected App in Salesforce
2. Enable OAuth with appropriate scopes
3. Note the Consumer Key and Consumer Secret
4. Generate security token

### PostgreSQL

```sql
-- Enable logical replication
ALTER SYSTEM SET wal_level = logical;

-- Create replication user
CREATE USER replication_user WITH REPLICATION PASSWORD 'repl_password';

-- Grant access to tables
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replication_user;
```

## Verification Commands

```bash
# Check all pods
kubectl get pods -A

# Check ArgoCD
kubectl get pods -n argocd

# Check Kafka
kubectl get pods -n kafka
kubectl get kafka -n kafka

# Check Kafka Connect
kubectl get kafkaconnect -n kafka
kubectl get kafkaconnector -n kafka

# Check connector status
kubectl get kafkaconnector -n kafka -o wide

# Describe specific connector
kubectl describe kafkaconnector oracle-source-connector -n kafka
kubectl describe kafkaconnector salesforce-source-connector -n kafka
kubectl describe kafkaconnector postgresql-source-connector -n kafka
```

## Connector Status Check

```bash
# Get Kafka Connect pod
kubectl get pods -n kafka -l app.kubernetes.io/name=kafka-connect

# Check REST API
kubectl exec -n kafka <connect-pod> -- curl -s http://localhost:8083/connectors
kubectl exec -n kafka <connect-pod> -- curl -s http://localhost:8083/connectors/<connector-name>/status
```

## Troubleshooting

### Oracle Connectors

```bash
# Check LogMiner availability
SELECT status FROM v$logmnr_session;

# Verify supplemental logging
SELECT supplemental_log_data_min, supplemental_log_data_all FROM v$database;
```

### PostgreSQL Connectors

```bash
# Check replication status
SELECT * FROM pg_stat_replication;

# Check publications
SELECT * FROM pg_publication;

# Check subscriptions
SELECT * FROM pg_subscription;
```

### Salesforce Connectors

```bash
# Verify API access
curl -s https://login.services.salesforce.com/services/oauth2/token \
  -d grant_type=password \
  -d client_id=<consumer-key> \
  -d client_secret=<consumer-secret> \
  -d username=<username> \
  -d password=<password><security-token>
```

## Security Considerations

- Change all default passwords in production
- Use secrets management (Vault, Sealed Secrets)
- Enable network policies
- Use TLS for all communications
- Implement proper RBAC
- Store database credentials in Kubernetes secrets

## Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Camel-K Documentation](https://camel.apache.org/camel-k/)
- [Strimzi Documentation](https://strimzi.io/docs/)
- [Debezium Connectors](https://debezium.io/documentation/reference/stable/connectors/)
- [Kafka Connect JDBC](https://docs.confluent.io/kafka-connect-jdbc/current/)
- [Confluent Salesforce Connector](https://docs.confluent.io/kafka-connect-salesforce/current/)
