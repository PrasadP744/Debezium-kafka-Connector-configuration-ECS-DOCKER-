# Debezium-kafka-Connector-configuration-ECS-DOCKER-
It monitors databases and streams real-time changes (like inserts, updates, deletes) into systems such as Kafka, Pulsar, or Redpanda — so other services can react instantly

# Debezium CDC Setup Guide

## Overview
This guide covers the setup and configuration of Debezium for Change Data Capture (CDC) from MySQL to Kafka. The Docker Compose configuration is maintained separately in the repository.

---

## Prerequisites

- Docker and Docker Compose installed
- MySQL database with binary logging enabled
- Kafka cluster running and accessible
- Network `dab-nw` created (`docker network create dab-nw`)

---

## MySQL Configuration

### 1. Enable Binary Logging

Ensure your MySQL instance has binary logging enabled. Add the following to your MySQL configuration:

```ini
[mysqld]
server-id = 223344
log_bin = mysql-bin
binlog_format = ROW
binlog_row_image = FULL
expire_logs_days = 10
```

### 2. Create Debezium User

Create a dedicated MySQL user with the necessary permissions:

```sql
CREATE USER 'debezium'@'%' IDENTIFIED BY 'dbz';

GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT 
ON *.* TO 'debezium'@'%';

FLUSH PRIVILEGES;
```

---

## Connector Configuration

### Creating a Connector File

Each database schema requires a separate connector configuration file. Create a JSON file (e.g., `mysql-connector.json`) with the following structure:

```json
{
  "name": "mysql-hp-archive-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "dev-mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "223345",
    "topic.prefix": "mysql_devhp",
    "database.include.list": "hp_archive",
    "table.include.list": "hp_archive.pp_items,hp_archive.po_revisions",
    "include.schema.changes": "true",
    "schema.history.internal.kafka.bootstrap.servers": "kafka1:9093",
    "schema.history.internal.kafka.topic": "schema-changes.hp_archive",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter.schemas.enable": "false",
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "mysql_devhp\\..*",
    "transforms.route.replacement": "mysql-devhp"
  }
}
```

### Configuration Parameters Explained

| Parameter | Description |
|-----------|-------------|
| `name` | Unique identifier for the connector |
| `database.hostname` | MySQL container name or hostname |
| `database.port` | MySQL port (container port) |
| `database.server.id` | Unique ID for this MySQL connection (must be unique across all connectors) |
| `topic.prefix` | Prefix for Kafka topics created by this connector |
| `database.include.list` | Comma-separated list of databases to monitor |
| `table.include.list` | Specific tables to monitor (format: `database.table`) |
| `schema.history.internal.kafka.topic` | Kafka topic for schema change history |

### Creating Multiple Connectors

For each schema, create a separate connector file with:
- Unique `name`
- Unique `database.server.id`
- Unique `topic.prefix`
- Unique `schema.history.internal.kafka.topic`

---

## Connector Management Commands

### Register/Enable a Connector

```bash
curl -X POST http://localhost:8085/connectors \
  -H "Content-Type: application/json" \
  -d @mysql-connector.json
```

### Check Connector Status

```bash
curl http://localhost:8085/connectors/mysql-hp-archive-connector/status
```

### List All Connectors

```bash
curl http://localhost:8085/connectors
```

### Get Connector Configuration

```bash
curl http://localhost:8085/connectors/mysql-hp-archive-connector
```

### Delete a Connector

```bash
curl -X DELETE http://localhost:8085/connectors/mysql-hp-archive-connector
```

### Pause a Connector

```bash
curl -X PUT http://localhost:8085/connectors/mysql-hp-archive-connector/pause
```

### Resume a Connector

```bash
curl -X PUT http://localhost:8085/connectors/mysql-hp-archive-connector/resume
```

### Restart a Connector

```bash
curl -X POST http://localhost:8085/connectors/mysql-hp-archive-connector/restart
```

---

## Kafka Topic Management

### Manual Topic Creation

If automatic topic creation fails, manually create the schema history topic:

```bash
# Access Kafka container
docker exec -it <kafka-container-name> bash

# Create schema history topic
kafka-topics --create \
  --topic schema-changes.hp_archive \
  --bootstrap-server kafka1:9093 \
  --partitions 1 \
  --replication-factor 3 \
  --config cleanup.policy=delete \
  --config retention.ms=-1 \
  --config min.insync.replicas=2 \
  --config segment.bytes=1073741824
```

### List Topics

```bash
kafka-topics --list --bootstrap-server kafka1:9093
```

### Describe Topic

```bash
kafka-topics --describe --topic mysql-devhp --bootstrap-server kafka1:9093
```

### Consume Messages (Testing)

```bash
kafka-console-consumer \
  --bootstrap-server kafka1:9093 \
  --topic mysql-devhp \
  --from-beginning
```

---

## Troubleshooting

### Check Debezium Logs

```bash
docker logs debezium-connect -f
```

### Common Issues

#### 1. Connection Refused to MySQL
- Ensure MySQL container is on the same Docker network (`dab-nw`)
- Verify MySQL hostname is correct
- Check MySQL is accepting connections from Debezium user

#### 2. Topic Creation Failures
- Manually create the schema history topic (see Kafka Topic Management section)
- Check Kafka cluster is healthy and accessible
- Verify replication factor matches available Kafka brokers

#### 3. Binlog Errors
- Ensure binary logging is enabled in MySQL
- Verify Debezium user has `REPLICATION SLAVE` and `REPLICATION CLIENT` privileges
- Check binlog format is set to `ROW`

#### 4. Connector Failed State
```bash
# Check detailed error
curl http://localhost:8085/connectors/mysql-hp-archive-connector/status

# Check connector configuration
curl http://localhost:8085/connectors/mysql-hp-archive-connector

# Restart the connector
curl -X POST http://localhost:8085/connectors/mysql-hp-archive-connector/restart
```

---

## Monitoring

### Health Check Endpoints

```bash
# Debezium Connect health
curl http://localhost:8085/

# Connector plugins
curl http://localhost:8085/connector-plugins

# Connector tasks
curl http://localhost:8085/connectors/mysql-hp-archive-connector/tasks
```

---

## Best Practices

1. **Server IDs**: Ensure each connector has a unique `database.server.id`
2. **Topic Naming**: Use consistent and descriptive `topic.prefix` values
3. **Table Filtering**: Use `table.include.list` to monitor only necessary tables
4. **Schema History**: Always configure `schema.history.internal.kafka.topic` for production
5. **Resource Limits**: Set appropriate CPU/memory limits in Docker Compose
6. **Monitoring**: Implement monitoring for connector status and lag
7. **Backups**: Regularly backup connector configurations

---

## Performance Tuning

### Connector Level

```json
{
  "config": {
    "max.batch.size": "2048",
    "max.queue.size": "8192",
    "poll.interval.ms": "1000",
    "snapshot.mode": "initial"
  }
}
```

### Snapshot Modes

- `initial` - Perform initial snapshot, then continue with CDC
- `schema_only` - Only capture schema, no data snapshot
- `never` - Never perform snapshot, only CDC
- `when_needed` - Perform snapshot only if no offset exists

---

## Security Considerations

1. **Database Credentials**: Use environment variables or secrets management
2. **Network Isolation**: Keep Debezium on a private network
3. **SSL/TLS**: Enable SSL for MySQL and Kafka connections in production
4. **Access Control**: Limit Debezium API access to authorized users only

---

## Additional Resources

- [Debezium Documentation](https://debezium.io/documentation/)
- [Debezium MySQL Connector](https://debezium.io/documentation/reference/connectors/mysql.html)
- [Kafka Connect REST API](https://docs.confluent.io/platform/current/connect/references/restapi.html)
