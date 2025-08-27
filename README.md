## Real-Time Auction Bidding with Kafka, Flask, MySQL, CSV, and Hive

This project is a small end-to-end streaming demo that shows how user inputs from a web UI are published to Kafka and consumed by multiple independent services for different sinks: MySQL (OLTP), CSV (file export), and Hive (analytics).

### Architecture
- **Flask producer** (`app.py`): Web form to submit bids. Publishes JSON messages to Kafka topic `auction` using Confluent JSON Serializer and Schema Registry.
- **Consumers (sinks)**:
  - `db_writer1.py` / `db_writer2.py`: Insert bids into MySQL table `test.bid`.
  - `file_writer.py`: Append bids to `output.csv`.
  - `hive_writer.py`: Insert bids into Hive table via ODBC (optional).
- **Schema**: JSON Schema stored in Confluent Schema Registry under subject `auction-value` ensures producer/consumer compatibility.

### Tech Stack
- Python 3, Flask (Jinja, Bootstrap)
- Apache Kafka (Confluent Python client)
- Confluent Schema Registry (JSON Schema)
- MySQL (mysql-connector-python)
- CSV sink
- Apache Hive via ODBC (pyodbc, optional)
- Auth: SASL_SSL + PLAIN for Kafka

---

## Prerequisites
1. Python 3.8+
2. Kafka cluster and Schema Registry (Confluent Cloud or self-hosted)
   - Kafka bootstrap server, API key/secret
   - Schema Registry URL, API key/secret
   - Topic `auction` created
3. MySQL accessible (local or remote)
   - Database `test`
   - Table `bid`
4. (Optional) Hive with HiveServer2 and the Cloudera Hive ODBC Driver installed

### Create the MySQL database/table
```sql
CREATE DATABASE IF NOT EXISTS test;
USE test;
CREATE TABLE IF NOT EXISTS bid (
  name VARCHAR(100),
  price INT,
  bid_ts DATETIME
);
```

### JSON Schema to register (subject: `auction-value`)
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Bid",
  "type": "object",
  "properties": {
    "name": { "type": "string" },
    "price": { "type": "integer" },
    "bid_ts": { "type": "string" }
  },
  "required": ["name", "price", "bid_ts"],
  "additionalProperties": false
}
```

---

## Setup
From the project root:
```powershell
python -m venv .venv
.venv\Scripts\activate
pip install -U pip
pip install flask confluent-kafka mysql-connector-python pyodbc
```

Configure Kafka and Schema Registry credentials in `config.py`:
```python
API_KEY = 'your API key'
ENDPOINT_SCHEMA_URL  = 'https://your-schema-registry-url'
API_SECRET_KEY = 'your api secret'
BOOTSTRAP_SERVER = 'your-bootstrap-server:9092'
SECURITY_PROTOCOL = 'SASL_SSL'
SSL_MECHANISM = 'PLAIN'
SCHEMA_REGISTRY_API_KEY = 'your sr key'
SCHEMA_REGISTRY_API_SECRET = 'your sr secret'
```

Update MySQL credentials in:
- `app.py` (for reading current highest bid)
- `db_writer1.py` / `db_writer2.py` (for inserts)

If using Hive, set connection info in `hive_writer.py` (host, port, user, password, database) and ensure the ODBC driver is installed.

---

## How to Run
Open two or more terminals (PowerShell) in the project directory.

### 1) Start one or more consumers (sinks)
Pick any of the following:
```powershell
# MySQL sink
python db_writer1.py

# OR another MySQL sink in a separate terminal
python db_writer2.py

# CSV sink
python file_writer.py

# Hive sink (optional)
python hive_writer.py
```

### 2) Start the web app (producer)
```powershell
python app.py
```
Open your browser at `http://127.0.0.1:5000/` and submit bids. The page displays the highest bid from the MySQL table. Consumers will process events from Kafka and write to their respective sinks.

---

## Project Structure
```
app.py                 # Flask producer: form -> Kafka (topic: auction)
config.py              # Kafka & Schema Registry config values
db_writer1.py          # Kafka consumer -> MySQL (test.bid)
db_writer2.py          # Kafka consumer -> MySQL (test.bid)
file_writer.py         # Kafka consumer -> output.csv
hive_writer.py         # Kafka consumer -> Hive via ODBC (optional)
templates/index.html   # Simple Bootstrap form and highest bid display
```

---

## Troubleshooting
- Producer errors about schema: ensure subject `auction-value` exists and matches the JSON sent.
- Authentication/authorization errors: verify `config.py` values and `SECURITY_PROTOCOL='SASL_SSL'`, `SSL_MECHANISM='PLAIN'` for Confluent Cloud.
- MySQL insert failures: check credentials/host, ensure table exists, and that `price` is an integer.
- No messages consumed: confirm the topic name is `auction` and that the consumer group is running; verify network connectivity to Kafka and Schema Registry.
- Hive errors: ensure ODBC driver installed and 64-bit compatibility with your Python; verify HiveServer2 connectivity.

---

## Extending the Project (nice resume wins)
- Add `.env` config and structured logging.
- Dockerize services and use docker-compose for local Kafka/MySQL/Schema Registry.
- Real-time leaderboard with WebSockets/SSE.
- Retry with backoff and dead-letter topic for poison messages.
- Batch inserts and connection pooling for higher throughput.
- CI tests with testcontainers for Kafka/MySQL.

---

## License
MIT (adjust as needed).


