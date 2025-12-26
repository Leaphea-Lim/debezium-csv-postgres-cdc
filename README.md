# Debezium CSV to PostgreSQL CDC Pipeline

A complete data pipeline solution using Kafka Connect with Debezium to capture and stream CSV data into PostgreSQL database. This project demonstrates Change Data Capture (CDC) patterns with secure SSL/TLS communication between all components.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Testing the Pipeline](#testing-the-pipeline)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [Additional Resources](#additional-resources)

## 🎯 Overview

This project implements a data ingestion pipeline that:
- Monitors a directory for CSV files
- Automatically ingests CSV data into Kafka topics
- Streams data from Kafka into PostgreSQL database
- Uses Avro for schema evolution and data serialization
- Implements SSL/TLS encryption for secure communication
- Provides a web UI for monitoring and management

**Key Technologies:**
- Apache Kafka (3-node cluster with KRaft mode)
- Debezium Kafka Connect
- Confluent Schema Registry
- PostgreSQL 16
- Kafka UI for monitoring

## 🏗 Architecture

```
CSV Files → Kafka Connect (SpoolDir) → Kafka Topics → JDBC Sink Connector → PostgreSQL
                                            ↓
                                    Schema Registry (Avro)
```

**Components:**
- **3 Kafka Brokers**: Distributed message streaming platform
- **Schema Registry**: Manages Avro schemas for data serialization
- **Debezium Kafka Connect**: Runs source and sink connectors
- **PostgreSQL**: Target database for data persistence
- **Kafka UI**: Web interface for monitoring (port 18000)

## ✅ Prerequisites

- Docker and Docker Compose installed
- At least 8GB RAM available for Docker
- Ports available: 5168, 29093, 39093, 49093, 8081, 8083, 18000
- Basic understanding of Kafka Connect concepts

## 📁 Project Structure

```
debezium-csv-postgres-cdc/
├── docker-compose.yml                    # Main orchestration file
├── Dockerfile                            # Custom Debezium Connect image
├── generate-ssl-certs.sh                 # SSL certificate generation script
├── secrets/                              # SSL certificates and credentials
│   ├── kafka.server.keystore.jks
│   ├── kafka.server.truststore.jks
│   ├── ca.crt
│   ├── kafka_keystore_creds
│   ├── kafka_ssl_key_creds
│   └── kafka_truststore_creds
├── config/                               # Kafka configuration files
├── csv-data/                            # CSV file input directory
│   ├── processed/                       # Successfully processed files
│   └── error/                           # Failed file processing
├── confluentinc-kafka-connect-jdbc-10.8.4/     # JDBC connector
├── jcustenborder-kafka-connect-spooldir-2.0.66/ # CSV SpoolDir connector
└── README.md
```

## 🚀 Quick Start

### Step 1: Generate SSL Certificates

Navigate to the project directory and generate SSL certificates:

```bash
cd debezium-csv-postgres-cdc
chmod +x generate-ssl-certs.sh
./generate-ssl-certs.sh
```

This creates all necessary SSL certificates in the `secrets/` directory for secure communication.

### Step 2: Start the Stack

Build and start all services:

```bash
docker compose up -d --build
```

Wait 2-3 minutes for all services to initialize. Check service health:

```bash
docker compose ps
```

All services should show status as "Up" or "healthy".

### Step 3: Verify Services

- **Kafka UI**: http://localhost:18000 (username: `admin`, password: `admin-secret`)
- **Kafka Connect**: http://localhost:8083
- **Schema Registry**: http://localhost:8081
- **PostgreSQL**: `localhost:5168` (username: `postgres`, password: `postgres`)

## ⚙️ Configuration

### Create CSV Source Connector

The SpoolDir connector monitors the `/csv-data` directory for new CSV files.

**Basic Configuration (Auto Schema Detection):**

```bash
curl -X PUT http://localhost:8083/connectors/csv-source-basic/config \
  -H "Content-Type: application/json" \
  -d '{
    "connector.class": "com.github.jcustenborder.kafka.connect.spooldir.SpoolDirCsvSourceConnector",
    "topic": "orders_csv_topic",
    "input.path": "/csv-data",
    "finished.path": "/csv-data/processed",
    "error.path": "/csv-data/error",
    "input.file.pattern": ".*\\.csv",
    "schema.generation.enabled": "true",
    "csv.first.row.as.header": "true"
  }'
```

**Advanced Configuration (With Type Casting):**

For better data type handling, use the Cast transformation:

```bash
curl -X PUT http://localhost:8083/connectors/csv-source-typed/config \
  -H "Content-Type: application/json" \
  -d '{
    "connector.class": "com.github.jcustenborder.kafka.connect.spooldir.SpoolDirCsvSourceConnector",
    "topic": "orders_csv_typed",
    "input.path": "/csv-data",
    "finished.path": "/csv-data/processed",
    "error.path": "/csv-data/error",
    "input.file.pattern": ".*\\.csv",
    "schema.generation.enabled": "true",
    "csv.first.row.as.header": "true",
    "transforms": "castTypes",
    "transforms.castTypes.type": "org.apache.kafka.connect.transforms.Cast$Value",
    "transforms.castTypes.spec": "order_id:int32,customer_id:int32,order_total_usd:float32"
  }'
```

**Note**: Adjust the `transforms.castTypes.spec` based on your CSV columns and desired data types.

### Create PostgreSQL Sink Connector

Stream data from Kafka to PostgreSQL:

```bash
curl -X PUT http://localhost:8083/connectors/postgres-sink/config \
  -H "Content-Type: application/json" \
  -d '{
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "table.name.format": "orders_table",
    "connection.password": "postgres",
    "tasks.max": "1",
    "topics": "orders_csv_topic",
    "auto.evolve": "true",
    "connection.user": "postgres",
    "auto.create": "true",
    "connection.url": "jdbc:postgresql://cdc-postgres:5432/postgres",
    "insert.mode": "upsert",
    "pk.mode": "record_value",
    "pk.fields": "order_id"
  }'
```

**Important Configuration Parameters:**
- `topics`: Must match the topic name from your source connector
- `table.name.format`: Name of the PostgreSQL table to create
- `auto.create`: Automatically creates the table if it doesn't exist
- `auto.evolve`: Updates table schema when new columns are added
- `insert.mode`: Use `upsert` for update/insert, or `insert` for append-only
- `pk.fields`: Primary key column(s) for upsert operations

### Connector Management

**List all connectors:**
```bash
curl http://localhost:8083/connectors
```

**Check connector status:**
```bash
curl http://localhost:8083/connectors/csv-source-basic/status
```

**Pause a connector:**
```bash
curl -X PUT http://localhost:8083/connectors/csv-source-basic/pause
```

**Resume a connector:**
```bash
curl -X PUT http://localhost:8083/connectors/csv-source-basic/resume
```

**Delete a connector:**
```bash
curl -X DELETE http://localhost:8083/connectors/csv-source-basic
```

## 🧪 Testing the Pipeline

### Step 1: Prepare Sample CSV

Create a sample CSV file `orders.csv`:

```csv
order_id,customer_id,product_name,order_total_usd,order_date
1001,501,Laptop,1299.99,2024-01-15
1002,502,Mouse,29.99,2024-01-16
1003,501,Keyboard,79.99,2024-01-17
```

### Step 2: Place CSV in Monitored Directory

Copy the CSV file into the container's monitored directory:

```bash
docker cp orders.csv debezium-kafka-connect:/csv-data/
```

**What happens next:**

1. **Detection**: The SpoolDir connector detects the new file within seconds
2. **Processing**: Each row is read and published to the Kafka topic
3. **Movement**: Upon successful processing, `orders.csv` is moved to `/csv-data/processed/`
4. **Streaming**: The JDBC Sink connector reads from the Kafka topic
5. **Persistence**: Data is inserted/updated in the PostgreSQL `orders_table`

If processing fails, the file is moved to `/csv-data/error/` instead.

### Step 3: Verify Data Flow

**Check Kafka Topic (via Kafka UI):**
1. Navigate to http://localhost:18000
2. Login with `admin` / `admin-secret`
3. Click on "Topics" in the left menu
4. Find your topic (e.g., `orders_csv_topic`)
5. Click "Messages" to view the data

**Check PostgreSQL Database:**

Using `psql`:
```bash
docker exec -it cdc-postgres psql -U postgres -d postgres
```

Query the data:
```sql
SELECT * FROM orders_table;
```

**Using Database Tools (DataGrip, DBeaver, pgAdmin):**
- Host: `localhost`
- Port: `5168`
- Database: `postgres`
- Username: `postgres`
- Password: `postgres`

### Step 4: Test Schema Evolution

Add a new column to your CSV:

```csv
order_id,customer_id,product_name,order_total_usd,order_date,shipping_address
1004,503,Monitor,399.99,2024-01-18,123 Main St
```

Place this file in `/csv-data/`. The connector will:
1. Detect the new column
2. Update the Avro schema in Schema Registry
3. Automatically add the column to PostgreSQL table (if `auto.evolve: true`)

## 📊 Monitoring

### Kafka UI Dashboard

Access the Kafka UI at http://localhost:18000

**Available Features:**
- View all Kafka topics and their messages
- Monitor consumer groups and lag
- Check connector status and tasks
- Browse Schema Registry schemas
- View broker configurations and metrics

### Kafka Connect REST API

**Check cluster info:**
```bash
curl http://localhost:8083/
```

**List connector plugins:**
```bash
curl http://localhost:8083/connector-plugins
```

**View connector configuration:**
```bash
curl http://localhost:8083/connectors/csv-source-basic
```

**Check task status:**
```bash
curl http://localhost:8083/connectors/csv-source-basic/tasks/0/status
```

### Logs

**View connector logs:**
```bash
docker logs -f debezium-kafka-connect
```

**View Kafka broker logs:**
```bash
docker logs -f kafka-1
```

**View PostgreSQL logs:**
```bash
docker logs -f cdc-postgres
```

## 🔧 Troubleshooting

### Issue: Connector fails to start

**Check connector status:**
```bash
curl http://localhost:8083/connectors/csv-source-basic/status
```

**Common causes:**
- Incorrect topic name or configuration
- Missing dependencies in connector classpath
- SSL certificate issues

**Solution**: Check logs and verify configuration parameters.

### Issue: CSV file not being processed

**Possible reasons:**
1. File doesn't match the `input.file.pattern` (default: `.*\.csv`)
2. File is locked by another process
3. Connector is paused

**Solution:**
```bash
# Check connector status
curl http://localhost:8083/connectors/csv-source-basic/status

# Resume if paused
curl -X PUT http://localhost:8083/connectors/csv-source-basic/resume

# Check if file is in error directory
docker exec debezium-kafka-connect ls -la /csv-data/error/
```

### Issue: Data not appearing in PostgreSQL

**Checklist:**
1. Verify source connector is running and producing messages
2. Check if topic has messages (via Kafka UI)
3. Verify sink connector is running
4. Check sink connector configuration (topic name, connection URL)

**Debug steps:**
```bash
# Check if topic exists and has data
docker exec kafka-1 kafka-topics --bootstrap-server localhost:19093 \
  --command-config /etc/kafka/config/client-ssl.properties --list

# Check sink connector status
curl http://localhost:8083/connectors/postgres-sink/status
```

### Issue: Schema compatibility errors

**Symptom**: Connector logs show schema compatibility errors

**Solution:**
- Use `auto.evolve: true` in sink connector for schema changes
- Or manually update the target table schema
- Check Schema Registry for registered schemas:
  ```bash
  curl http://localhost:8081/subjects
  ```

### Issue: SSL/TLS connection errors

**Solution:**
1. Regenerate certificates: `./generate-ssl-certs.sh`
2. Restart all services: `docker compose restart`
3. Verify certificate files exist in `./secrets/` directory

## 📚 Additional Resources

### Tutorials and Documentation

- **Original Tutorial**: [Loading CSV Data into Kafka by Robin Moffatt](https://rmoff.net/2020/06/17/loading-csv-data-into-kafka/)
- **Debezium Documentation**: https://debezium.io/documentation/
- **Kafka Connect Documentation**: https://kafka.apache.org/documentation/#connect
- **Confluent Schema Registry**: https://docs.confluent.io/platform/current/schema-registry/

### Connector Documentation

- **SpoolDir Connector**: https://github.com/jcustenborder/kafka-connect-spooldir
- **JDBC Sink Connector**: https://docs.confluent.io/kafka-connect-jdbc/current/

### Related Projects

- Debezium PostgreSQL CDC: For capturing direct database changes
- Kafka Streams: For real-time data transformations
- ksqlDB: For stream processing with SQL

## 📄 License

This project is provided as-is for educational and development purposes.

---

**Note**: This setup is configured for development and testing. For production use, consider:
- Using separate certificates for each component
- Implementing proper secret management
- Adding monitoring and alerting
- Configuring appropriate replication factors
- Setting up proper backup and disaster recovery