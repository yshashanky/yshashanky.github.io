Yes, I get your point. The easiest option is to give you the whole page in **clean Markdown-style formatting** that Confluence generally converts well when pasted into the editor: headings, bullets, tables, code blocks, bold text, and notes with minimal manual work.

Copy everything below starting from the title and paste it directly into your Confluence page.

---

# Kafka Viewer UI

## Overview

Kafka Viewer UI is a lightweight, read-only utility for viewing and inspecting Kafka topic messages through a browser-based interface.

Kafka Viewer can be used to:

- Connect to unsecured or secured Kafka clusters.
- Browse available Kafka topics.
- Load Kafka messages using different loading modes.
- Filter messages using single, OR, or AND conditions.
- Deserialize Confluent Avro messages using Schema Registry.
- View topic statistics.
- Download loaded or filtered messages as JSON.

> **Important:** Kafka Viewer is read-only. It does not publish messages or commit consumer offsets.

---

## Prerequisites

Before installing Kafka Viewer:

- Install **Python** using the approved **Dev CLI**.
- Ensure `pip` is available.

Verify the installation:

```bash
python --version
pip --version
```

---

## Install Kafka Viewer

Install Kafka Viewer version `0.4.0`:

```bash
pip install kafka-viewer==0.4.0
```

### Verify the Installed Version

Run:

```bash
pip show kafka-viewer
```

Verify that the output contains:

```text
Version: 0.4.0
```

---

## Start Kafka Viewer

Kafka Viewer requires a properties file containing the Kafka connection configuration.

Run:

```bash
kafka-viewer --config C:\configs\kafka.properties
```

Replace the path with the actual location of your properties file.

Example:

```bash
kafka-viewer --config C:\Users\<username>\configs\kafka.properties
```

---

## Kafka Configuration

Use one of the following configurations depending on the Kafka environment.

### 1. Unsecured Kafka

Example `kafka.properties`:

```properties
kafka.bootstrap.servers=broker-host:9092
```

For multiple brokers:

```properties
kafka.bootstrap.servers=broker1:9092,broker2:9092,broker3:9092
```

Run Kafka Viewer:

```bash
kafka-viewer --config C:\configs\kafka.properties
```

---

### 2. Secured Kafka

For Kafka clusters using SSL certificates:

```properties
kafka.bootstrap.servers=broker-host:9093
kafka.security.protocol=SSL

kafka.ssl.ca.file=C:\certs\ca.pem
kafka.ssl.cert.file=C:\certs\client-cert.pem
kafka.ssl.key.file=C:\certs\client-key.pem

kafka.ssl.check.hostname=true
kafka.ssl.endpoint.identification.algorithm=https
```

Update the broker and certificate paths according to your environment.

| Property | Description |
| --- | --- |
| `kafka.bootstrap.servers` | Kafka broker address or addresses. |
| `kafka.security.protocol` | Security protocol used to connect to Kafka. |
| `kafka.ssl.ca.file` | CA certificate used to validate the Kafka broker certificate. |
| `kafka.ssl.cert.file` | Client certificate used for authentication. |
| `kafka.ssl.key.file` | Private key associated with the client certificate. |
| `kafka.ssl.check.hostname` | Enables or disables Kafka broker hostname verification. |
| `kafka.ssl.endpoint.identification.algorithm` | Endpoint identification algorithm used during TLS validation. |

---

### 3. Secured Kafka with Confluent Avro

Use the following configuration when Kafka is secured and messages use Confluent Avro with Schema Registry.

```properties
kafka.bootstrap.servers=broker-host:9093
kafka.security.protocol=SSL

kafka.ssl.ca.file=C:\certs\ca.pem
kafka.ssl.cert.file=C:\certs\client-cert.pem
kafka.ssl.key.file=C:\certs\client-key.pem

kafka.ssl.check.hostname=true
kafka.ssl.endpoint.identification.algorithm=https

schema.registry.url=https://schema-registry-host:8082
schema.registry.ssl.verify=false
```

Kafka Viewer uses Schema Registry to deserialize supported Confluent Avro messages.

> **Security Note:** `schema.registry.ssl.verify=false` disables TLS certificate verification for Schema Registry. Use this only when required and approved for your environment.

---

## Starting the Kafka Viewer UI

Run:

```bash
kafka-viewer --config C:\configs\kafka.properties
```

Once Kafka Viewer starts, the terminal displays URLs similar to:

```text
Local URL:   http://localhost:8501
Network URL: http://<machine-ip>:8501
```

### Local URL

The **Local URL** opens Kafka Viewer only on the machine where Kafka Viewer is running.

Example:

```text
http://localhost:8501
```

Use this for normal individual usage.

### Network URL

The **Network URL** can be used to access the running Kafka Viewer from another system that can reach your machine over the same approved network or VDI environment.

Example:

```text
http://<machine-ip>:8501
```

This can be useful when another team member on the same network needs to access the running Kafka Viewer instance.

> **Important:** Share the Network URL only within an approved and trusted corporate network.

---

> **Windows Defender / Firewall Notice**
>
> When Kafka Viewer is started for the first time, Windows may display a Windows Defender Firewall or security pop-up asking whether Python/Streamlit should be allowed to communicate over the network.
>
> Allow or unblock the application for the appropriate approved corporate network when required. This prompt should normally appear only during the first-time setup.

---

## Kafka Viewer UI

Once Kafka Viewer has started, open the **Local URL** in your browser.

Most controls in the UI are self-explanatory. The sections below explain the options that require additional context.

---

## Connection Status

The top section displays the Kafka connection status.

Available actions:

- **Test / Refresh Connection** — verifies or refreshes the Kafka connection.
- **Refresh Topics** — reloads the available Kafka topics.

---

## Topic Selection

Select the Kafka topic that you want to inspect.

The available topics are retrieved from the Kafka cluster configured in your properties file.

---

## Consumer Group ID

Some Kafka environments require consumer group IDs to use an approved or whitelisted prefix.

### Group ID Prefix

Enter the **whitelisted Group ID Prefix** approved for your Kafka environment.

Example:

```text
my-application-kafka-viewer-
```

Kafka Viewer can then generate a temporary consumer group ID using this prefix.

Example generated value:

```text
my-application-kafka-viewer-<temporary-id>
```

Use **Generate Temporary Group ID** where applicable.

> **Note:** The Consumer Group ID is required for consuming Kafka records. Kafka Viewer does not commit consumer offsets.

---

## Loading Modes

Kafka Viewer provides multiple ways to load records.

### Latest

Loads the latest requested number of messages.

Example:

```text
Message Count: 100
```

Kafka Viewer will return up to the latest 100 messages.

When a Message Filter is applied, Kafka Viewer may need to inspect more records than the requested Message Count to find matching messages.

The default maximum scan limit is:

```text
5000
```

This can optionally be changed in the properties file:

```properties
kafka.viewer.filter.scan.max.records=20000
```

For example:

```properties
kafka.viewer.filter.scan.max.records=40000
```

allows Kafka Viewer to inspect up to 40,000 records while searching for filtered matches.

Kafka Viewer stops scanning when any of the following occurs:

- The requested number of matching records is found.
- The configured scan limit is reached.
- All available retained records have been exhausted.

The configured scan limit is a **global limit across all topic partitions**.

> **Note:** Increasing the scan limit may increase processing time and resource usage.

---

### From Beginning

Loads messages starting from the beginning of the retained Kafka data.

---

### From Date/Time

Loads messages beginning from the selected date and time.

---

### Date/Time Range

Loads messages between the selected start and end date/time.

The range follows:

```text
start <= message timestamp < end
```

---

## Message Filtering

Kafka Viewer supports case-insensitive literal filtering against message content.

If Confluent Avro deserialization is enabled and successful, filtering is applied to the **deserialized message**.

Otherwise, filtering is applied to the raw message value.

Filtering does not apply to Kafka metadata such as:

- Partition
- Offset
- Timestamp
- Consumer Group ID

### Single Value Filter

Example:

```text
payment
```

This returns messages containing `payment`.

---

### OR Filter

Use `?` between values for an **OR** condition.

Example:

```text
payment?failed?timeout
```

The message is returned if it contains any of the following:

```text
payment
OR
failed
OR
timeout
```

If the same message matches multiple values, it is returned only once.

---

### AND Filter

Use `&` between values for an **AND** condition.

Example:

```text
payment&failed&timeout
```

The message must contain all of the following:

```text
payment
AND
failed
AND
timeout
```

within the same Kafka record.

---

### Filter Rules

Do not mix `?` and `&` in the same filter expression.

Not supported:

```text
payment?failed&timeout
```

Use either:

```text
payment?failed?timeout
```

or:

```text
payment&failed&timeout
```

Filtering is case-insensitive.

---

## Topic Statistics

Kafka Viewer displays the following topic statistics:

- **Total Records**
- **Published Today**
- **Last 1 Hour**
- **Partitions**
- **Latest Record Timestamp**

Use **Refresh Statistics** to refresh the values.

---

## Load Messages

After configuring:

- Topic
- Consumer Group ID
- Loading Mode
- Message Count or Date/Time
- Optional Message Filter

click:

**Load Messages**

Kafka Viewer loads the selected records without committing Kafka consumer offsets.

Use **Clear Loaded Messages** to clear the currently displayed messages.

---

## Message Details

Loaded Kafka records are displayed under **Message Details**.

Depending on the configured topic and properties, messages may be shown as:

- Raw/String data
- Deserialized Confluent Avro data

If Avro deserialization fails for a record, Kafka Viewer retains and displays the available raw payload instead of failing the entire message load.

---

## Download Messages

Loaded or filtered records can be downloaded using the **JSON download** option.

This can be useful for:

- Offline analysis
- Debugging
- Comparing records
- Reviewing specific Kafka messages

> **Important:** Handle downloaded Kafka data according to applicable internal data-handling and security requirements.

---

## Quick Start Example

### Step 1 — Install Kafka Viewer

```bash
pip install kafka-viewer==0.4.0
```

### Step 2 — Verify Version

```bash
pip show kafka-viewer
```

Confirm:

```text
Version: 0.4.0
```

### Step 3 — Create a Properties File

Example:

```properties
kafka.bootstrap.servers=broker-host:9093
kafka.security.protocol=SSL
kafka.ssl.ca.file=C:\certs\ca.pem
kafka.ssl.cert.file=C:\certs\client-cert.pem
kafka.ssl.key.file=C:\certs\client-key.pem
kafka.ssl.check.hostname=true
kafka.ssl.endpoint.identification.algorithm=https
```

### Step 4 — Start Kafka Viewer

```bash
kafka-viewer --config C:\configs\kafka.properties
```

### Step 5 — Open the UI

Open the **Local URL** displayed in the terminal.

### Step 6 — Configure and Load Messages

1. Select the Kafka topic.
2. Enter the approved **Group ID Prefix**.
3. Generate or enter the Consumer Group ID.
4. Select the required Loading Mode.
5. Enter Message Count or Date/Time criteria.
6. Add a Message Filter if required.
7. Click **Load Messages**.

---

## Important Usage Notes

- Kafka Viewer is a **read-only Kafka inspection utility**.
- Kafka Viewer does **not commit consumer offsets**.
- Kafka Viewer does **not publish messages**.
- Use only approved Kafka connection details.
- Use only approved/whitelisted Consumer Group ID prefixes.
- Keep private keys, certificates, credentials, and properties files in approved secure locations.
- Share the Network URL only within an approved trusted network.
- Increasing the filtered scan limit can increase processing time.
- Use `schema.registry.ssl.verify=false` only where specifically required and approved.
