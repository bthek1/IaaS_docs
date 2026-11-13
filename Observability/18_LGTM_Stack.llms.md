# LGTM stack

The **OTEL + LGTM** stack is a **powerful observability solution** that combines:

Author

Benedict Thekkel

## **1. Introduction**

### **Key Benefits**

✅ **End-to-End Observability** → Tracks logs, metrics, and traces from applications.\
✅ **Scalability** → Can handle large-scale distributed systems.\
✅ **Open-Source and Vendor-Neutral** → Works with many cloud and on-prem solutions.\
✅ **Seamless Grafana Integration** → Unified dashboards for logs, metrics, and traces.

------------------------------------------------------------------------

## **2. Overview of the OTEL + LGTM Stack**

| **Component** | **Purpose** |
|----|----|
| **OTEL (OpenTelemetry)** | Standardized **tracing, metrics, and logs collection**. |
| **Loki** | **Log aggregation** system for storing and querying logs efficiently. |
| **Grafana** | **Visualization** and monitoring dashboards for logs, metrics, and traces. |
| **Tempo** | **Distributed tracing** backend for tracking requests across services. |
| **Mimir** | **High-performance metrics storage** for Prometheus-like queries. |

------------------------------------------------------------------------

## **3. Architecture**

### **How OTEL + LGTM Works**

    +------------------+       +------------------+       +------------------+       +------------------+
    |   Application    | --->  | OpenTelemetry    | --->  |   Loki (Logs)    | --->  | Grafana (UI)    |
    |   (Microservices)|       | Collector        |       |   Tempo (Traces) |       | Dashboards      |
    |                 |       | (Traces, Metrics, Logs) |   |   Mimir (Metrics) |       | Alerts, Queries |
    +------------------+       +------------------+       +------------------+       +------------------+

------------------------------------------------------------------------

## **4. Setting Up OTEL + LGTM Stack**

### **4.1 Install Docker and Docker Compose**

Ensure **Docker** and **Docker Compose** are installed:

``` bash
sudo apt update
sudo apt install docker docker-compose -y
```

Verify installation:

``` bash
docker --version
docker-compose --version
```

------------------------------------------------------------------------

## **5. Deploy OTEL + LGTM Stack Using Docker Compose**

Create a **`docker-compose.yml`** file:

``` yaml
version: "3.8"

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-collector
    volumes:
      - ./otel-config.yaml:/etc/otel-collector-config.yaml
    command: ["--config", "/etc/otel-collector-config.yaml"]
    ports:
      - "4317:4317"  # gRPC receiver
      - "4318:4318"  # HTTP receiver
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - /var/log:/var/log
      - ./promtail-config.yaml:/etc/promtail/promtail.yaml
    command: -config.file=/etc/promtail/promtail.yaml
    depends_on:
      - loki

  mimir:
    image: grafana/mimir:latest
    container_name: mimir
    ports:
      - "9009:9009"

  tempo:
    image: grafana/tempo:latest
    container_name: tempo
    ports:
      - "3200:3200"
    command: [ "-config.file=/etc/tempo.yaml" ]

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - loki
      - mimir
      - tempo

volumes:
  grafana_data:
```

------------------------------------------------------------------------

## **6. Configure OpenTelemetry Collector**

Create an **otel-config.yaml** file:

``` yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

exporters:
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
  prometheus:
    endpoint: "0.0.0.0:9464"
  otlp:
    endpoint: tempo:3200
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp]
    metrics:
      receivers: [otlp]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      exporters: [loki]
```

------------------------------------------------------------------------

## **7. Configure Promtail (Log Collector)**

Create **promtail-config.yaml**:

``` yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system_logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*.log
```

------------------------------------------------------------------------

## **8. Start the Stack**

Run the following command:

``` bash
docker-compose up -d
```

Verify running containers:

``` bash
docker ps
```

------------------------------------------------------------------------

## **9. Access Grafana Dashboards**

1.  Open Grafana: **`http://localhost:3000`**
2.  Login: **Username: `admin`, Password: `admin`**
3.  Add **Data Sources**:
    - **Loki** → For logs (`http://loki:3100`)
    - **Prometheus** → For metrics (`http://otel-collector:9464`)
    - **Tempo** → For traces (`http://tempo:3200`)

------------------------------------------------------------------------

## **10. Querying Data in Grafana**

### **Logs Query (Loki)**

    {job="varlogs"}

Find logs related to a process:

    {job="varlogs"} |= "error"

### **Metrics Query (Mimir)**

CPU Usage:

    rate(node_cpu_seconds_total[5m])

### **Traces Query (Tempo)**

Track requests by service:

    trace_id="12345abcde"

------------------------------------------------------------------------

## **11. Example Application with OpenTelemetry**

To send logs, traces, and metrics from an **application**, use OpenTelemetry SDK.

### **Install OpenTelemetry Python SDK**

``` bash
pip install opentelemetry-sdk opentelemetry-exporter-otlp
```

### **Instrument an Application**

``` python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace.export import SimpleSpanProcessor

trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer_provider().get_tracer(__name__)

exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)
trace.get_tracer_provider().add_span_processor(SimpleSpanProcessor(exporter))

with tracer.start_as_current_span("example-trace"):
    print("Tracing this execution.")
```

Run it:

``` bash
python app.py
```

------------------------------------------------------------------------

## **12. Summary**

| **Component** | **Purpose**                                          |
|---------------|------------------------------------------------------|
| **OTEL**      | Collects logs, metrics, and traces from applications |
| **Loki**      | Stores logs for efficient querying                   |
| **Grafana**   | Visualizes metrics, logs, and traces                 |
| **Tempo**     | Handles distributed tracing                          |
| **Mimir**     | Stores and queries Prometheus-style metrics          |

This setup provides a **complete observability solution** for **monitoring, logging, and tracing applications** with **scalability and flexibility**.

------------------------------------------------------------------------

Back to top
