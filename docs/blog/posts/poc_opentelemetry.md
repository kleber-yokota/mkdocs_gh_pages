# Proof of Concept: OpenTelemetry with Python and Docker

This project is a **technical Proof of Concept (PoC)** to explore how [OpenTelemetry](https://opentelemetry.io/) works for tracing, logging, and metrics collection in Python applications.  
The goal is to understand its architecture and capabilities to enable deeper future analyses and observability improvements.

## Why This PoC?

Modern systems require **deep visibility** into application behavior to detect issues, optimize performance, and improve reliability.  
This PoC was created to:

- **Learn** how OpenTelemetry collects and exports telemetry data (traces, metrics, logs).
- **Experiment** with local telemetry ingestion and visualization using Grafana's **otel-lgtm** stack.
- **Lay the foundation** for applying observability practices to more complex, production-grade systems in the future.

By understanding these concepts now, it will be easier to build **smarter monitoring, alerting, and analysis pipelines** later.

## Benefits of Using OpenTelemetry

- **Standardization:** One open-source standard for observability across vendors and technologies.
- **Flexibility:** Easily export telemetry data to many different backends (Grafana, Datadog, Jaeger, Prometheus, etc.).
- **Rich Telemetry:** Collect traces, logs, and metrics in a **unified** and **correlated** way.
- **Vendor Neutrality:** Avoid lock-in by instrumenting once and choosing/exporting to any compatible backend.
- **Extensibility:** OpenTelemetry supports advanced use cases like baggage propagation, contextual data, and auto-instrumentation.

## Possible Applications of OpenTelemetry

In future projects, OpenTelemetry can be used to:

- **Monitor microservices** in real-time (track service-to-service communications).
- **Optimize resource usage** (CPU, memory, network) and detect bottlenecks automatically.
- **Correlate failures** across distributed systems.
- **Create intelligent alerting systems** based on real telemetry patterns.
- **Improve system reliability** by proactively detecting anomalies before users are affected.
- **Enable better post-mortem analysis** by combining traces, logs, and metrics into a single timeline.

## About OpenTelemetry

[OpenTelemetry](https://opentelemetry.io/) is an open-source observability framework that provides APIs, SDKs, and tools to instrument, generate, collect, and export telemetry data (logs, metrics, and traces).
It is a CNCF project and aims to standardize the way software systems are monitored.

In this project, OpenTelemetry is used to:

- **Trace** ETL steps (`extract`, `transform`, and `load`) using spans.
- **Log** important messages during the ETL flow.
- **Measure** CPU and RAM usage dynamically during execution.

All telemetry data is sent to a local OpenTelemetry backend deployed via Docker Compose.

## Project Structure

- **Python ETL script** with OpenTelemetry tracing, logging, and metrics.
- **Docker Compose** file to run the observability backend stack using Grafana's [otel-lgtm](https://grafana.com/blog/2023/09/13/announcing-otel-lgtm-the-easiest-way-to-get-logs-metrics-and-traces-from-opentelemetry/).
- **pyproject.toml** to manage Python dependencies.

## Running the Project

### 1. Start the Observability Backend

```bash
docker-compose up -d
```

This will launch the following components:

- **Grafana UI** (port `3000`)
- **Tempo** (tracing backend)
- **Loki** (logging backend)
- **Mimir** (metrics backend)
- **OTLP receiver** (`4317` for gRPC, `4318` for HTTP)

Grafana credentials:
- **User:** admin
- **Password:** admin

Access Grafana: [http://localhost:3000](http://localhost:3000)

### 2. Install Python Dependencies

```bash
uv pip install --system
```

This will install the necessary libraries from `pyproject.toml`.

### 3. Run the ETL Script

```bash
python main.py
```

This script will:
- Simulate CPU-intensive work by finding large prime numbers.
- Log each ETL step (extract, transform, load).
- Export traces, metrics, and logs to the OpenTelemetry backend.

## Key Parts of the Code

### OpenTelemetry Setup

```python
from opentelemetry import trace, metrics, _logs
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk._logs import LoggerProvider, LoggingHandler
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.exporter.otlp.proto.grpc._log_exporter import OTLPLogExporter

# Tracer, Meter, and Logger providers configured
```

### ETL Simulation Functions

```python
def extract():
    with tracer.start_as_current_span("extract") as span:
        stress_cpu()
        logger.info("Extract step completed")
        span.set_status(Status(StatusCode.OK))

def transform():
    with tracer.start_as_current_span("transform") as span:
        stress_cpu()
        logger.info("Transform step completed")
        span.set_status(Status(StatusCode.OK))

def load():
    with tracer.start_as_current_span("load") as span:
        time.sleep(random.uniform(1, 2))
        logger.info("Load step completed")
        span.set_status(Status(StatusCode.OK))
```

### CPU and RAM Metrics

```python
def cpu(span_id):
    def callback(options):
        process = psutil.Process()
        yield Observation(process.cpu_percent(interval=0.5), {"span_id": span_id})
    return callback

def ram(span_id):
    def callback(options):
        process = psutil.Process()
        yield Observation(process.memory_info().rss / (1024 * 1024), {"span_id": span_id})
    return callback
```

## Observability Architecture

```plaintext
[Python ETL Script] -> [OpenTelemetry SDK] -> [OTLP gRPC Exporter] -> [otel-lgtm stack (Tempo, Loki, Mimir, Grafana)]
```

## Future Improvements
- Add attributes and events to spans.
- Include exception tracking.
- Explore baggage propagation across services.
- Use a distributed system (multiple microservices) for a more realistic setup.

## Conclusion

This PoC gives a foundational understanding of how OpenTelemetry can instrument a Python application and export telemetry data to observability tools.

---

**Repository:** https://github.com/your-username/poc-opentelemetry

**Author:** Your Name

**License:** MIT
