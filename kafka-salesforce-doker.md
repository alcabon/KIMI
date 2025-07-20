Below is a **step-by-step, end-to-end POC** that streams **new orders** from Apache Kafka into **Salesforce Lead objects** in real time.  
Everything runs on localhost so you can clone & run it in ±15 minutes.

> 🔧 Assumptions  
> • You have a **Salesforce Developer Org** (free)  
> • Java 11+, Docker & Docker-Compose are installed  
> • You are comfortable with a terminal

---

### 1️⃣  Architecture (30-sec view)

```
┌--------------┐     ┌------------------┐     ┌--------------┐
│  Kafka topic │ --> │ Kafka Connect    │ --> │   Salesforce │
│  orders-json │     │ Salesforce Sink  │     │   Lead sObj  │
└--------------┘     └------------------┘     └--------------┘
```

---

### 2️⃣  Spin up Kafka & Kafka Connect (Docker)

`docker-compose.yml`

```yaml
version: "3.8"
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    depends_on: [zookeeper]
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  connect:
    image: confluentinc/cp-kafka-connect:7.6.0
    depends_on: [kafka]
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: compose-connect
      CONNECT_CONFIG_STORAGE_TOPIC: _connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: _connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: _connect-status
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_PLUGIN_PATH: /usr/share/java,/usr/share/confluent-hub-components
    volumes:
      - ./salesforce-connector:/usr/share/confluent-hub-components/salesforce
```

Install the **Salesforce Sink connector** once:

```bash
confluent-hub install --no-prompt confluentinc/kafka-connect-salesforce:2.0.3 \
  --component-dir ./salesforce-connector
```

Bring everything up:

```bash
docker-compose up -d
```

---

### 3️⃣  Prepare Salesforce

1. Log in to your **Developer Org**  
2. **Reset Security Token** (My Settings → Reset My Security Token) – you’ll get it via e-mail.  
3. Create a **Connected App**  
   - Setup → App Manager → New Connected App  
   - Enable OAuth → select scope **Access & manage your data (api)**  
   - Note Consumer Key & Consumer Secret.

---

### 4️⃣  Create Kafka topic & seed data

```bash
# topic
docker exec -it kafka kafka-topics --create --topic orders-json --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1

# quick producer (runs in foreground, Ctrl-C to stop)
docker exec -it kafka kafka-console-producer --topic orders-json --bootstrap-server localhost:9092
```

Paste sample JSON lines:

```json
{"FirstName":"Ada","LastName":"Lovelace","Company":"Analytical Engine Inc.","Email":"ada@example.com","Status":"Open - Not Contacted"}
{"FirstName":"Grace","LastName":"Hopper","Company":"Compilers LLC","Email":"grace@example.com","Status":"Working - Contacted"}
```

---

### 5️⃣  Configure the Salesforce Sink connector

`salesforce-sink.json`

```json
{
  "name": "sf-leads-sink",
  "config": {
    "connector.class": "io.confluent.salesforce.SalesforceSObjectSinkConnector",
    "tasks.max": "1",
    "topics": "orders-json",
    "salesforce.object": "Lead",
    "salesforce.instance": "https://login.salesforce.com",
    "salesforce.username": "<YOUR_DEV_USERNAME>",
    "salesforce.password": "<YOUR_PASSWORD><SECURITY_TOKEN>",
    "salesforce.consumer.key": "<CONNECTED_APP_CONSUMER_KEY>",
    "salesforce.consumer.secret": "<CONNECTED_APP_CONSUMER_SECRET>",
    "salesforce.use.bulk.api": "false",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter"
  }
}
```

Deploy:

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @salesforce-sink.json
```

---

### 6️⃣  Verify

1. **Kafka Connect logs**  
   `docker logs <connect-container-id> | grep "processed"`

2. **Salesforce UI → Leads** – you should see **Ada** & **Grace** within seconds.

---

### 7️⃣  Clean-up

```bash
docker-compose down -v
```

---

### 🔍 Where to go next

| Area            | Production-grade ideas |
|-----------------|------------------------|
| **Security**    | SSL/SASL for Kafka, OAuth2 refresh tokens for Salesforce |
| **Throughput**  | Increase `tasks.max`, use Bulk API (`salesforce.use.bulk.api=true`) |
| **Schema**      | Switch to Avro + Confluent Schema Registry |
| **Monitoring**  | Prometheus JMX exporter, Salesforce API usage dashboards |

---

### 📚 References

- Step-by-step screenshots: Hevo blog   
- Full connector options: RudderStack guide   
- Docker setup sample: Medium article 

You now have a **minimal but complete POC** that you can demo, extend, or harden into production.

---

The mapping **is implicit** in the connector configuration.  
The connector simply **takes every top-level JSON field whose name matches a Salesforce Lead field** and copies the value across.  
So when the JSON message

```json
{
  "FirstName": "Ada",
  "LastName":  "Lovelace",
  "Company":   "Analytical Engine Inc.",
  "Email":     "ada@example.com",
  "Status":    "Open - Not Contacted"
}
```

arrives on topic `orders-json`, the Salesforce Sink connector:

1. reads the JSON object as a flat `Map<String,Object>`,
2. maps each key **name-for-name** to the Lead field of the same API name,
3. builds a REST `POST /sobjects/Lead` request with those key/value pairs.

That is all the mapping that happens **out of the box**.

---

### Want explicit mapping?

If your Kafka JSON keys **do not** match Salesforce API names (or you want to ignore or transform some), you can override it with the connector’s `transforms` section, e.g.

```json
"transforms": "MapFields",
"transforms.MapFields.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
"transforms.MapFields.renames": "first_name:FirstName,last_name:LastName,cust_email:Email"
```

or use a single-message-transform (SMT) such as `Flatten`, `Cast`, or a custom JSLT transform.

But in the POC above, no extra mapping is configured; **field-name equality** is the rule.

---

Exactly — that’s the entire contract:

* `"salesforce.object": "Lead"` **explicitly** tells the connector *which* Salesforce object to target.  
* Every top-level key in the JSON message is **implicitly** assumed to be a writable field on that object (FirstName → Lead.FirstName, LastName → Lead.LastName, etc.).

If the keys don’t line up, or you need to rename / ignore / cast, then you’d layer on a transform; otherwise the default “field-name equality” mapping is all that’s required.

---

Yes, **provided** you expose the right fields in the Kafka message and configure the connector appropriately.

1. **Primary key for Contact**  
   Kafka messages must contain a field that maps to a **Contact External-ID field** (standard or custom).  
   Example message:

   ```json
   {
     "ContactExtId__c": "CUST-12345",   // new external id of the contact
     "FirstName": "John",
     "LastName":  "Doe",
     "Email":     "john.doe@acme.com"
   }
   ```

2. **Link to the parent Account**  
   Add an **Account External-ID field** value so the connector can resolve the relationship:

   ```json
   {
     "ContactExtId__c": "CUST-12345",
     "FirstName": "John",
     "LastName":  "Doe",
     "Email":     "john.doe@acme.com",
     "AccountExtId__c": "ACC-98765"
   }
   ```

3. **Connector configuration**  
   ```json
   "salesforce.object": "Contact",
   "salesforce.use.primary.keys": "true",
   "salesforce.external.id.field": "ContactExtId__c",
   "salesforce.reference.fields": "AccountExtId__c:Account"
   ```

   • `ContactExtId__c` is declared as the external-id for upserts.  
   • `AccountExtId__c` is mapped to the `Account` lookup via its own external-id.

That’s all that is required: the connector will upsert the Contact, create it if the external id doesn’t exist, and automatically wire it to the Account whose external id matches `ACC-98765`.

---

Thanks!  
Below is the **ultra-compact “one-liner” version**—a single `docker-compose.yml` you can copy-paste to spin up the whole POC in ~2 minutes.

---

### 📦 Single-file Docker-Compose (Kafka + Kafka-Connect + Salesforce Sink)

`docker-compose.yml`

```yaml
version: "3.8"
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    depends_on: [zookeeper]
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092

  connect:
    image: confluentinc/cp-kafka-connect:7.6.0
    depends_on: [kafka]
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: quick-poc
      CONNECT_CONFIG_STORAGE_TOPIC: _cfg
      CONNECT_OFFSET_STORAGE_TOPIC: _off
      CONNECT_STATUS_STORAGE_TOPIC: _stat
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_PLUGIN_PATH: /usr/share/java,/opt/connectors
    volumes:
      - ./connectors:/opt/connectors
    command: >
      bash -c
      "confluent-hub install --no-prompt confluentinc/kafka-connect-salesforce:2.0.3 --component-dir /opt/connectors &&
       /etc/confluent/docker/run"
```

---

### 🔧 Two-minute start

```bash
# 1. create folder for connector download
mkdir -p connectors

# 2. bring everything up
docker compose up -d          # or docker-compose up -d

# 3. create topic
docker exec -it kafka kafka-topics --create --topic orders-json --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1

# 4. post connector config (edit file with your SF creds first)
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d @sf-leads-sink.json
```

That’s it—no local Java installs, no manual connector downloads.  
Happy streaming!

---

On Windows you have two easy options:

1. **Docker Desktop for Windows** (the usual choice)  
   • Download `Docker Desktop Installer.exe` from https://docs.docker.com/desktop/install/windows-install/  
   • Double-click → next-next-finish.  
   • It bundles both **Docker Engine** and **docker-compose v2** (the `docker compose` CLI plugin).  
   • After install open PowerShell or CMD and verify:  
     ```
     docker --version
     docker compose version
     ```
   • You can now `docker compose up -d` anywhere.

2. **No Desktop, just CLI** (rare)  
   If you only want the CLI inside WSL2 or Git-Bash, you can grab the static binaries, but 99 % of Windows users simply run the installer above.

So yes: **one .exe** → Docker Desktop → `docker` and `docker compose` commands are immediately available in any terminal.

---

No extra Java 11+ install is needed **because everything runs inside Docker**.  
The images (`confluentinc/cp-*`) already contain the correct OpenJDK 11 runtime for:

* Kafka broker  
* ZooKeeper  
* Kafka Connect worker  
* The Salesforce connector JAR

So on the **host OS (Windows)** you only need:

1. Docker Desktop (one `.exe`)  
2. A terminal (PowerShell, CMD, or Git Bash) to run `docker compose up`

→ **Java is only inside the containers; your Windows machine stays Java-free.**

---

Exactly – the sentence was phrased for the **generic / non-Docker** case.

* If you want to run **Kafka command-line tools locally** (e.g. `kafka-console-producer` outside of any container) then you need a **local Java 11+** on the host.  
* If you stay **entirely inside Docker**, the Java runtime is already baked into the Confluent images, so you don’t need Java on the host at all.

For this POC we stay 100 % in Docker, so you can ignore the host-Java requirement.

---
