# Asterisk Observability with OpenTelemetry and Grafana

**How to actually see what's happening inside your Asterisk servers. OpenTelemetry as the collection layer, Prometheus for storage, Grafana for dashboards, and distributed tracing to follow a call from SIP INVITE to agent headset. Built from production VICIdial clusters pushing 200K+ daily calls.**

---

I've been running Asterisk in production since the 1.4 days. For most of that time, "monitoring" meant SSH into the box, run `asterisk -rx "core show channels"`, squint at the output, and hope that the number of active channels looked about right. Maybe check `/var/log/asterisk/full` when something broke. Maybe not.

That stopped being acceptable around the time we crossed 50,000 daily calls across a 4-server cluster. When a SIP trunk goes down at 2 PM on a Tuesday and 300 agents go idle, you need to know in seconds, not whenever someone notices the real-time report looks weird and pings you on Slack.

This guide covers the full observability stack for Asterisk: metrics collection with OpenTelemetry, storage in Prometheus, visualization in Grafana, and distributed tracing for individual call flows. If you're running VICIdial, everything here applies — VICIdial's call processing is just Asterisk dialplan execution under the hood, and all the telemetry surfaces the same way.

---

## Why OpenTelemetry Instead of Just Prometheus

You could skip OpenTelemetry entirely. Install `prometheus-node-exporter` on your Asterisk box, write a script that scrapes `asterisk -rx` output into Prometheus metrics, and call it done. I've done exactly that. It works. It's also fragile, custom, and doesn't scale.

OpenTelemetry (OTel) gives you three things that roll-your-own monitoring doesn't:

**Vendor-neutral collection.** The OTel Collector speaks StatsD, Prometheus, OTLP, syslog, and dozens of other formats. Asterisk's built-in `res_statsd` module pushes metrics via StatsD. AMI events can be forwarded as structured logs. You don't have to write custom parsers — you configure receivers.

**Processing pipelines.** OTel lets you filter, transform, aggregate, and route telemetry data before it hits your backend. Want to drop debug-level events but keep warnings? Want to add a `cluster_name` attribute to every metric? Want to sample 10% of traces for non-error calls? All configurable in the collector.

**Multi-backend export.** Send metrics to Prometheus, traces to Jaeger or Tempo, and logs to Loki — from one collector instance. If you ever want to switch from Prometheus to Mimir or from Jaeger to Tempo, you change one exporter config. Nothing on the Asterisk side changes.

That said, if you have a single Asterisk box running 5,000 calls a day, a Prometheus scraper script is probably fine. OTel shines when you have multiple servers, multiple signal types (metrics + traces + logs), or when you're tired of maintaining custom scripts.

---

## Architecture Overview

Here's what we're building:

```
┌─────────────────────────────────────────────────────┐
│  Asterisk Server                                     │
│                                                       │
│  res_statsd ──→ OTel Collector ──→ Prometheus        │
│                    (sidecar)                          │
│  AMI Events ──→ ami-otel-bridge ──→ OTel Collector   │
│                                       │               │
│  CDR/CEL ──→ MySQL ──→ mysqld_exporter ──→ Prometheus│
│                                                       │
└───────────────────────────────────────────────────────┘
         │                    │                │
         ▼                    ▼                ▼
   ┌──────────┐    ┌────────────────┐   ┌──────────┐
   │Prometheus│    │ Jaeger / Tempo  │   │   Loki   │
   └────┬─────┘    └───────┬────────┘   └────┬─────┘
        │                  │                  │
        └──────────┬───────┘──────────────────┘
                   │
              ┌────▼─────┐
              │  Grafana  │
              └──────────┘
```

Components:
- **res_statsd** — Asterisk's built-in StatsD module. Emits metrics on channel counts, endpoint status, bridge operations, and more.
- **OTel Collector** — Runs as a sidecar process on the Asterisk server. Receives StatsD from res_statsd, processes it, exports to Prometheus.
- **ami-otel-bridge** — A small script that reads Asterisk Manager Interface (AMI) events and converts them to OTel spans/logs.
- **mysqld_exporter** — Exports MySQL metrics for CDR/CEL table monitoring.
- **Prometheus** — Time-series database. Stores all metrics.
- **Grafana** — Dashboards and alerting.
- **Jaeger/Tempo** — Distributed tracing backend for call flow traces.

Let's build each layer.

---

## Step 1: Install the OpenTelemetry Collector

The OTel Collector runs on each Asterisk server as a systemd service.

```bash
# Download the latest stable release (check https://github.com/open-telemetry/opentelemetry-collector-releases)
OTEL_VERSION="0.96.0"
curl -L "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v${OTEL_VERSION}/otelcol-contrib_${OTEL_VERSION}_linux_amd64.tar.gz" \
  -o /tmp/otelcol.tar.gz
tar xzf /tmp/otelcol.tar.gz -C /usr/local/bin/ otelcol-contrib
chmod +x /usr/local/bin/otelcol-contrib

# Verify
otelcol-contrib --version
```

Create the systemd unit:

```ini
# /etc/systemd/system/otelcol.service
[Unit]
Description=OpenTelemetry Collector
After=network.target

[Service]
Type=simple
User=otelcol
Group=otelcol
ExecStart=/usr/local/bin/otelcol-contrib --config=/etc/otelcol/config.yaml
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

Create the user and directories:

```bash
useradd --system --no-create-home --shell /usr/sbin/nologin otelcol
mkdir -p /etc/otelcol
chown otelcol:otelcol /etc/otelcol
```

---

## Step 2: Configure Asterisk's StatsD Module

Asterisk has had built-in StatsD support since version 13 through `res_statsd`. It's compiled in by default on most distributions but not loaded by default.

Enable it:

```ini
# /etc/asterisk/statsd.conf
[general]
enabled = yes
server = 127.0.0.1:8125    ; OTel Collector's StatsD receiver
prefix = asterisk           ; All metrics will be prefixed with "asterisk."
add_newline = no
```

Load the module:

```bash
asterisk -rx "module load res_statsd.so"
# Verify it's loaded
asterisk -rx "module show like statsd"
```

Output should show:

```
Module                         Description                              Use Count  Status
res_statsd.so                  StatsD client support                    0          Running
```

### What Metrics Does res_statsd Emit?

Once loaded, Asterisk pushes the following metrics as StatsD gauges and counters:

| Metric | Type | Description |
|--------|------|-------------|
| `asterisk.channels.count` | gauge | Current active channel count |
| `asterisk.channels.by_type.SIP` | gauge | Active SIP channels |
| `asterisk.channels.by_type.PJSIP` | gauge | Active PJSIP channels |
| `asterisk.channels.by_type.Local` | gauge | Active Local channels |
| `asterisk.endpoints.count` | gauge | Registered endpoints |
| `asterisk.endpoints.state.online` | gauge | Endpoints in online state |
| `asterisk.endpoints.state.offline` | gauge | Endpoints in offline state |
| `asterisk.bridges.count` | gauge | Active bridges |
| `asterisk.bridges.channels` | gauge | Channels in bridges |

These are updated every 10 seconds by default. For a busy system, that's fine. If you need sub-second resolution (you probably don't), you can adjust the interval in `statsd.conf`.

---

## Step 3: OTel Collector Configuration

Here's the collector config that receives StatsD from Asterisk and exports to Prometheus:

```yaml
# /etc/otelcol/config.yaml
receivers:
  # Receive StatsD metrics from res_statsd
  statsd:
    endpoint: "0.0.0.0:8125"
    aggregation_interval: 10s
    timer_histogram_mapping:
      - statsd_type: "timer"
        observer_type: "histogram"
        histogram:
          explicit:
            - 10
            - 25
            - 50
            - 100
            - 250
            - 500
            - 1000
            - 5000
            - 10000

  # Scrape host metrics (CPU, memory, disk, network)
  hostmetrics:
    collection_interval: 15s
    scrapers:
      cpu:
        metrics:
          system.cpu.utilization:
            enabled: true
      memory:
        metrics:
          system.memory.utilization:
            enabled: true
      disk: {}
      network: {}
      load: {}

  # Receive OTLP from custom instrumentation (ami-otel-bridge)
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"
      http:
        endpoint: "0.0.0.0:4318"

processors:
  # Add resource attributes to every metric
  resource:
    attributes:
      - key: service.name
        value: "asterisk"
        action: upsert
      - key: host.name
        from_attribute: ""
        action: upsert
      - key: cluster.name
        value: "vicidial-prod"
        action: upsert
      - key: server.role
        value: "dialer"
        action: upsert

  # Batch metrics to reduce export overhead
  batch:
    timeout: 10s
    send_batch_size: 1000

  # Memory limiter to prevent OOM
  memory_limiter:
    check_interval: 5s
    limit_mib: 256
    spike_limit_mib: 64

exporters:
  # Export metrics to Prometheus
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "asterisk"
    resource_to_telemetry_conversion:
      enabled: true

  # Export traces to Jaeger (or Tempo)
  otlp/jaeger:
    endpoint: "jaeger.monitoring.local:4317"
    tls:
      insecure: true

  # Export logs to Loki
  loki:
    endpoint: "http://loki.monitoring.local:3100/loki/api/v1/push"
    labels:
      attributes:
        service.name: "service_name"
        host.name: "hostname"

  # Debug output (disable in production)
  # debug:
  #   verbosity: detailed

service:
  pipelines:
    metrics:
      receivers: [statsd, hostmetrics]
      processors: [memory_limiter, resource, batch]
      exporters: [prometheus]
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [otlp/jaeger]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [loki]

  telemetry:
    logs:
      level: "warn"
    metrics:
      address: ":8888"
```

Start the collector:

```bash
systemctl daemon-reload
systemctl enable otelcol
systemctl start otelcol

# Verify it's running and receiving StatsD
curl -s http://localhost:8889/metrics | grep asterisk_channels
```

You should see Prometheus-formatted metrics:

```
# HELP asterisk_channels_count Current active channel count
# TYPE asterisk_channels_count gauge
asterisk_channels_count{cluster_name="vicidial-prod",host_name="dialer01",server_role="dialer"} 47
```

---

## Step 4: Custom Metrics via AMI

StatsD gives you the basics — channel counts, endpoint status, bridge counts. But for VICIdial-specific observability, you need more. The Asterisk Manager Interface (AMI) emits events for everything: new channels, hangups, DTMF, queue joins, agent status changes, you name it.

Here's a Python script that connects to AMI, listens for events, and pushes them to the OTel Collector as metrics and traces:

```python
#!/usr/bin/env python3
"""
ami_otel_bridge.py — Bridge AMI events to OpenTelemetry
Runs as a daemon alongside Asterisk.
"""

import socket
import time
import re
import os
from opentelemetry import metrics, trace
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# OTel setup
metric_exporter = OTLPMetricExporter(endpoint="localhost:4317", insecure=True)
metric_reader = PeriodicExportingMetricReader(metric_exporter, export_interval_millis=10000)
metrics.set_meter_provider(MeterProvider(metric_readers=[metric_reader]))
meter = metrics.get_meter("ami-bridge")

trace_exporter = OTLPSpanExporter(endpoint="localhost:4317", insecure=True)
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(BatchSpanProcessor(trace_exporter))
tracer = trace.get_tracer("ami-bridge")

# Metrics
calls_total = meter.create_counter("asterisk.calls.total", description="Total calls")
calls_active = meter.create_up_down_counter("asterisk.calls.active", description="Active calls")
calls_by_disposition = meter.create_counter("asterisk.calls.by_disposition", description="Calls by disposition")
sip_registrations = meter.create_up_down_counter("asterisk.sip.registrations", description="SIP registration events")
queue_callers = meter.create_up_down_counter("asterisk.queue.callers", description="Callers waiting in queue")
call_duration = meter.create_histogram("asterisk.call.duration_ms", description="Call duration in milliseconds")

# Track active call spans for distributed tracing
active_spans = {}

AMI_HOST = os.environ.get("AMI_HOST", "127.0.0.1")
AMI_PORT = int(os.environ.get("AMI_PORT", "5038"))
AMI_USER = os.environ.get("AMI_USER", "admin")
AMI_SECRET = os.environ.get("AMI_SECRET", "amp111")


def connect_ami():
    """Connect to AMI and authenticate."""
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(30)
    sock.connect((AMI_HOST, AMI_PORT))

    # Read banner
    sock.recv(1024)

    # Login
    login = (
        f"Action: Login\r\n"
        f"Username: {AMI_USER}\r\n"
        f"Secret: {AMI_SECRET}\r\n"
        f"Events: call,agent,cdr\r\n"
        f"\r\n"
    )
    sock.sendall(login.encode())
    response = sock.recv(4096).decode()
    if "Success" not in response:
        raise ConnectionError(f"AMI login failed: {response}")

    print(f"[ami-otel] Connected to AMI at {AMI_HOST}:{AMI_PORT}")
    return sock


def parse_event(raw):
    """Parse an AMI event into a dict."""
    event = {}
    for line in raw.strip().split("\r\n"):
        if ": " in line:
            key, value = line.split(": ", 1)
            event[key.strip()] = value.strip()
    return event


def handle_event(event):
    """Process an AMI event and emit OTel signals."""
    event_type = event.get("Event", "")

    if event_type == "Newchannel":
        channel = event.get("Channel", "unknown")
        calls_total.add(1, {"channel_type": channel.split("/")[0]})
        calls_active.add(1)

        # Start a trace span for this call
        uniqueid = event.get("Uniqueid", "")
        if uniqueid:
            span = tracer.start_span(
                "asterisk.call",
                attributes={
                    "asterisk.channel": channel,
                    "asterisk.uniqueid": uniqueid,
                    "asterisk.caller_id": event.get("CallerIDNum", ""),
                    "asterisk.context": event.get("Context", ""),
                    "asterisk.exten": event.get("Exten", ""),
                }
            )
            active_spans[uniqueid] = {
                "span": span,
                "start_time": time.time(),
            }

    elif event_type == "Hangup":
        calls_active.add(-1)
        uniqueid = event.get("Uniqueid", "")
        cause = event.get("Cause-txt", "Unknown")

        # End the trace span
        if uniqueid in active_spans:
            span_data = active_spans.pop(uniqueid)
            duration_ms = (time.time() - span_data["start_time"]) * 1000
            span_data["span"].set_attribute("asterisk.hangup_cause", cause)
            span_data["span"].set_attribute("asterisk.duration_ms", duration_ms)
            span_data["span"].end()
            call_duration.record(duration_ms, {"cause": cause})

    elif event_type == "AgentComplete":
        dispo = event.get("Reason", "unknown")
        calls_by_disposition.add(1, {"disposition": dispo})

    elif event_type == "PeerStatus":
        peer = event.get("Peer", "")
        status = event.get("PeerStatus", "")
        if status == "Registered":
            sip_registrations.add(1, {"peer": peer})
        elif status == "Unregistered":
            sip_registrations.add(-1, {"peer": peer})

    elif event_type == "Join":
        queue_callers.add(1, {"queue": event.get("Queue", "unknown")})

    elif event_type == "Leave":
        queue_callers.add(-1, {"queue": event.get("Queue", "unknown")})


def main():
    while True:
        try:
            sock = connect_ami()
            buffer = ""

            while True:
                data = sock.recv(4096).decode("utf-8", errors="replace")
                if not data:
                    raise ConnectionError("AMI connection lost")

                buffer += data

                # AMI events are separated by \r\n\r\n
                while "\r\n\r\n" in buffer:
                    raw_event, buffer = buffer.split("\r\n\r\n", 1)
                    if raw_event.strip():
                        event = parse_event(raw_event)
                        if "Event" in event:
                            handle_event(event)

        except Exception as e:
            print(f"[ami-otel] Error: {e}, reconnecting in 5s...")
            time.sleep(5)


if __name__ == "__main__":
    main()
```

Install dependencies and run as a service:

```bash
pip3 install opentelemetry-api opentelemetry-sdk \
    opentelemetry-exporter-otlp-proto-grpc

# Create systemd unit
cat > /etc/systemd/system/ami-otel-bridge.service << 'EOF'
[Unit]
Description=AMI to OpenTelemetry Bridge
After=asterisk.service otelcol.service

[Service]
Type=simple
User=asterisk
Environment=AMI_HOST=127.0.0.1
Environment=AMI_PORT=5038
Environment=AMI_USER=admin
Environment=AMI_SECRET=your_ami_password_here
ExecStart=/usr/bin/python3 /usr/local/bin/ami_otel_bridge.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable ami-otel-bridge
systemctl start ami-otel-bridge
```

Now you have two metric sources feeding the OTel Collector: `res_statsd` for Asterisk internals, and the AMI bridge for call-level events and distributed traces.

---

## Step 5: Prometheus Configuration

Prometheus needs to scrape the OTel Collector's Prometheus exporter endpoint:

```yaml
# /etc/prometheus/prometheus.yml (add to scrape_configs)
scrape_configs:
  - job_name: 'asterisk-otel'
    scrape_interval: 10s
    static_configs:
      - targets:
          - 'dialer01.internal:8889'
          - 'dialer02.internal:8889'
          - 'dialer03.internal:8889'
        labels:
          environment: 'production'

  # Also scrape the OTel Collector's own health metrics
  - job_name: 'otel-collector'
    scrape_interval: 30s
    static_configs:
      - targets:
          - 'dialer01.internal:8888'
          - 'dialer02.internal:8888'
          - 'dialer03.internal:8888'
```

### Recording Rules for Call Center KPIs

Raw metrics are useful, but derived metrics are where the value lives. Set up recording rules in Prometheus:

```yaml
# /etc/prometheus/rules/asterisk.yml
groups:
  - name: asterisk_kpis
    interval: 30s
    rules:
      # Calls per minute (cluster-wide)
      - record: asterisk:calls_per_minute
        expr: sum(rate(asterisk_calls_total[5m])) * 60

      # Average call duration (5-minute window)
      - record: asterisk:avg_call_duration_sec
        expr: |
          histogram_quantile(0.5,
            rate(asterisk_call_duration_ms_bucket[5m])
          ) / 1000

      # 95th percentile call duration
      - record: asterisk:p95_call_duration_sec
        expr: |
          histogram_quantile(0.95,
            rate(asterisk_call_duration_ms_bucket[5m])
          ) / 1000

      # Channel utilization per server (active / max)
      - record: asterisk:channel_utilization
        expr: |
          asterisk_channels_count /
          (asterisk_endpoints_state_online * 2)

      # SIP registration churn rate
      - record: asterisk:registration_churn_rate
        expr: |
          abs(rate(asterisk_sip_registrations[5m]))

      # Queue wait callers (cluster total)
      - record: asterisk:queue_callers_total
        expr: sum(asterisk_queue_callers)
```

Reload Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## Step 6: Grafana Dashboards

Now the fun part. Here's where you actually see things. If you haven't set up Grafana yet, our [VICIdial Grafana dashboard guide](https://vicistack.com/blog/vicidial-grafana-dashboards/) covers the basic installation.

### Dashboard 1: Cluster Overview

The single-pane-of-glass dashboard. This should be on a TV on the wall.

**Panel 1: Active Channels (Stat)**

```
Query: sum(asterisk_channels_count)
Thresholds: 0-100 green, 100-200 yellow, 200+ red
```

**Panel 2: Calls Per Minute (Time Series)**

```
Query: asterisk:calls_per_minute
Legend: {{host_name}}
```

**Panel 3: Channel Utilization by Server (Gauge)**

```
Query: asterisk:channel_utilization * 100
Legend: {{host_name}}
Min: 0, Max: 100
Thresholds: 0-70 green, 70-85 yellow, 85-100 red
```

**Panel 4: SIP Registrations (Stat + Sparkline)**

```
Query: sum(asterisk_endpoints_state_online)
```

**Panel 5: Call Duration Distribution (Heatmap)**

```
Query: sum(rate(asterisk_call_duration_ms_bucket[5m])) by (le)
Format: Heatmap
```

**Panel 6: Queue Callers Waiting (Time Series)**

```
Query: sum(asterisk_queue_callers) by (queue)
Legend: {{queue}}
Alert: if > 10 for 2 minutes
```

### Dashboard 2: SIP Health

This dashboard tells you when trunks are dying before your agents notice.

**Panel 1: Registration Status by Peer (Table)**

```
Query: asterisk_endpoints_state_online
Transform: Labels to fields
Columns: host_name, peer, value
Value mappings: 1 = "Online" (green), 0 = "Offline" (red)
```

**Panel 2: Registration Events Rate (Time Series)**

```
Query: rate(asterisk_sip_registrations[5m])
Legend: {{peer}}
```

A spike in registration events means phones are flapping — registering, dropping, re-registering. This usually indicates a network issue between the phone and Asterisk, or a DNS problem with the SIP registrar.

**Panel 3: Active Channels by Type (Pie Chart)**

```
Query: asterisk_channels_by_type
Legend: {{channel_type}}
```

In a healthy VICIdial system, you should see mostly PJSIP (or SIP) channels for agent phones and trunk calls, with some Local channels for internal routing. If you see IAX2 channels spiking, that's inter-server traffic in a cluster — normal during peak, but worth watching.

### Dashboard 3: Per-Server Deep Dive

For when you suspect a specific server is misbehaving:

```
Variables:
  - server: label_values(asterisk_channels_count, host_name)

Panels:
  1. CPU Usage:     system_cpu_utilization{host_name="$server"}
  2. Memory Usage:  system_memory_utilization{host_name="$server"}
  3. Channels:      asterisk_channels_count{host_name="$server"}
  4. Load Average:  system_cpu_load_average_5m{host_name="$server"}
  5. Network I/O:   rate(system_network_io_bytes_total{host_name="$server"}[5m])
  6. Disk I/O:      rate(system_disk_io_bytes_total{host_name="$server"}[5m])
```

The magic correlation: if CPU spikes at the same time channels spike, that's expected. If CPU spikes without a channel increase, something else is eating resources — check for runaway AGI scripts, a MySQL query from hell, or a cron job that shouldn't be running during peak hours.

---

## Step 7: Distributed Tracing for Call Flows

This is the part most Asterisk monitoring setups miss entirely. Metrics tell you *that* something happened. Traces tell you *why*.

A distributed trace follows a single call from the moment the SIP INVITE arrives through every dialplan step, AGI execution, queue wait, agent delivery, and hangup. When a caller reports "I waited 3 minutes and then got disconnected," you can pull up that exact call's trace and see every hop.

The AMI bridge script above creates spans for each call. To get the full picture, you need child spans for key events within a call:

```python
# Enhanced event handling with nested spans
def handle_dial_begin(event):
    """Track outbound dial attempts within a call."""
    uniqueid = event.get("Uniqueid", "")
    if uniqueid in active_spans:
        parent_span = active_spans[uniqueid]["span"]
        ctx = trace.set_span_in_context(parent_span)
        child = tracer.start_span(
            "asterisk.dial",
            context=ctx,
            attributes={
                "asterisk.dial.destination": event.get("DestChannel", ""),
                "asterisk.dial.dialstring": event.get("Dialstring", ""),
            }
        )
        active_spans[uniqueid]["dial_span"] = child


def handle_dial_end(event):
    """Complete the dial span with the result."""
    uniqueid = event.get("Uniqueid", "")
    if uniqueid in active_spans and "dial_span" in active_spans[uniqueid]:
        dial_span = active_spans[uniqueid].pop("dial_span")
        dial_span.set_attribute("asterisk.dial.status", event.get("DialStatus", ""))
        dial_span.end()


def handle_queue_join(event):
    """Track time spent in queue."""
    uniqueid = event.get("Uniqueid", "")
    if uniqueid in active_spans:
        parent_span = active_spans[uniqueid]["span"]
        ctx = trace.set_span_in_context(parent_span)
        child = tracer.start_span(
            "asterisk.queue.wait",
            context=ctx,
            attributes={
                "asterisk.queue.name": event.get("Queue", ""),
                "asterisk.queue.position": event.get("Position", ""),
                "asterisk.queue.count": event.get("Count", ""),
            }
        )
        active_spans[uniqueid]["queue_span"] = child


def handle_queue_leave(event):
    """End queue wait span."""
    uniqueid = event.get("Uniqueid", "")
    if uniqueid in active_spans and "queue_span" in active_spans[uniqueid]:
        queue_span = active_spans[uniqueid].pop("queue_span")
        queue_span.end()
```

With this instrumentation, a typical inbound call trace in Jaeger looks like:

```
[asterisk.call] ─── 145.2s total
 ├── [asterisk.dial] ─── 0.8s (to queue)
 ├── [asterisk.queue.wait] ─── 12.4s (INBOUND_SALES queue)
 ├── [asterisk.dial] ─── 1.2s (to agent SIP/agent42)
 └── [asterisk.call] ends ─── hangup cause: Normal Clearing
```

You can immediately see: 12.4 seconds in queue. Was that normal? What was the queue depth? Was the agent available or did we have to wait for one to go READY? The trace answers all of it.

---

## Step 8: Alerting

Dashboards are for humans staring at screens. Alerts are for 3 AM.

### Prometheus Alerting Rules

```yaml
# /etc/prometheus/rules/asterisk_alerts.yml
groups:
  - name: asterisk_alerts
    rules:
      # Trunk down — no channels for 2 minutes
      - alert: AsteriskTrunkDown
        expr: |
          asterisk_channels_count == 0
          and on(host_name)
          (time() - asterisk_channels_count offset 5m) > 300
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Asterisk server {{ $labels.host_name }} has zero active channels for 2+ minutes"
          description: "All trunks may be down. Check SIP registrations and carrier connectivity."

      # Channel exhaustion warning
      - alert: AsteriskChannelExhaustion
        expr: asterisk:channel_utilization > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Channel utilization above 85% on {{ $labels.host_name }}"
          description: "Server is approaching channel capacity. Current utilization: {{ $value | humanizePercentage }}"

      # Registration storm — phones flapping
      - alert: AsteriskRegistrationStorm
        expr: asterisk:registration_churn_rate > 5
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "High SIP registration churn on {{ $labels.host_name }}"
          description: "Phones are registering/unregistering rapidly. Possible network instability."

      # Queue backup — callers waiting too long
      - alert: AsteriskQueueBackup
        expr: sum(asterisk_queue_callers) by (queue) > 15
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Queue {{ $labels.queue }} has {{ $value }} callers waiting"
          description: "More than 15 callers in queue for 2+ minutes. Check agent availability."

      # No calls for 10 minutes during business hours
      - alert: AsteriskNoCalls
        expr: |
          asterisk:calls_per_minute == 0
          and hour() >= 9
          and hour() <= 17
          and day_of_week() >= 1
          and day_of_week() <= 5
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Zero calls per minute during business hours on {{ $labels.host_name }}"

      # AMI bridge disconnected
      - alert: AMIBridgeDown
        expr: up{job="asterisk-otel"} == 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "OTel Collector not reachable for {{ $labels.instance }}"
```

Wire these into Alertmanager to send to Slack, PagerDuty, email, or whatever your on-call system uses.

---

## Step 9: CDR-Based Metrics

The AMI bridge captures real-time events, but CDR (Call Detail Records) give you the complete picture after a call ends. VICIdial stores CDRs in MySQL, and Prometheus can query MySQL through `mysqld_exporter` — but honestly, for CDR analysis, you're better off with a dedicated query:

```sql
-- Calls per hour with quality metrics (run from a cron-based exporter)
SELECT
    DATE_FORMAT(calldate, '%Y-%m-%d %H:00:00') AS hour_bucket,
    COUNT(*) AS total_calls,
    AVG(duration) AS avg_duration,
    AVG(billsec) AS avg_billsec,
    SUM(CASE WHEN disposition = 'ANSWERED' THEN 1 ELSE 0 END) AS answered,
    SUM(CASE WHEN disposition = 'NO ANSWER' THEN 1 ELSE 0 END) AS no_answer,
    SUM(CASE WHEN disposition = 'BUSY' THEN 1 ELSE 0 END) AS busy,
    SUM(CASE WHEN disposition = 'FAILED' THEN 1 ELSE 0 END) AS failed
FROM cdr
WHERE calldate >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
GROUP BY hour_bucket
ORDER BY hour_bucket;
```

For a Prometheus-native approach, write a small exporter script that queries CDR data and exposes it as Prometheus metrics:

```python
#!/usr/bin/env python3
"""
cdr_exporter.py — Export Asterisk CDR metrics to Prometheus
Run as a service, scrape on :9101/metrics
"""

import time
import pymysql
from prometheus_client import start_http_server, Gauge, Counter, Histogram

# Metrics
cdr_calls_total = Counter('asterisk_cdr_calls_total', 'Total CDR records', ['disposition'])
cdr_duration = Histogram('asterisk_cdr_duration_seconds', 'Call duration from CDR',
                         buckets=[5, 10, 30, 60, 120, 300, 600, 1800])
cdr_asr = Gauge('asterisk_cdr_asr', 'Answer-seizure ratio (15 min window)')


def collect_cdr_metrics():
    conn = pymysql.connect(
        host='127.0.0.1',
        user='cdr_readonly',
        password='readonly_password',
        db='asterisk',
        charset='utf8mb4'
    )
    try:
        with conn.cursor() as cursor:
            # Recent calls by disposition
            cursor.execute("""
                SELECT disposition, COUNT(*), AVG(billsec)
                FROM cdr
                WHERE calldate >= DATE_SUB(NOW(), INTERVAL 15 MINUTE)
                GROUP BY disposition
            """)
            total_calls = 0
            answered_calls = 0
            for row in cursor.fetchall():
                disposition, count, avg_bill = row
                cdr_calls_total.labels(disposition=disposition).inc(count)
                total_calls += count
                if disposition == 'ANSWERED':
                    answered_calls += count

            if total_calls > 0:
                cdr_asr.set(answered_calls / total_calls)

    finally:
        conn.close()


if __name__ == '__main__':
    start_http_server(9101)
    while True:
        collect_cdr_metrics()
        time.sleep(60)
```

---

## Putting It All Together: The 3 AM Scenario

It's 3:17 AM. PagerDuty wakes you up: **AsteriskTrunkDown** on dialer02.

Before this setup, here's what you'd do: SSH into dialer02, run `asterisk -rx "sip show peers"`, stare at the output, grep through logs, try to figure out when it broke and why, call the carrier, wait on hold.

With the observability stack, here's what you do:

1. **Open Grafana on your phone.** The Cluster Overview dashboard shows dialer02 with zero channels. Dialer01 and dialer03 are healthy.

2. **Check the SIP Health dashboard.** You see that dialer02's SIP trunk to Carrier A went offline at 3:04 AM. Registration events show a rapid unregister/register cycle starting at 3:01 AM — three minutes of flapping before it gave up.

3. **Open the Prometheus alert timeline.** RegistrationStorm fired at 3:03 AM. TrunkDown fired at 3:06 AM. The flapping started before the trunk died — likely a network issue, not a carrier issue.

4. **Check host metrics.** dialer02's network I/O dropped to zero at 3:04 AM on eth1 (the trunk interface). CPU and memory are fine. It's a network link failure, not a server issue.

5. **Call the NOC**, not the carrier. The network link to the trunk VLAN is down. They fix it. Trunk comes back. Total incident resolution: 12 minutes instead of 45.

That's observability. Not dashboards for the sake of dashboards. Dashboards that tell you where to look and what to do.

---

## Performance Impact

The question everyone asks: does all this monitoring slow down Asterisk?

Short answer: no. Not measurably.

Longer answer: `res_statsd` adds roughly 0.1% CPU overhead on a server handling 100 concurrent channels. The AMI bridge script uses about 15MB of RAM and negligible CPU — it's event-driven, not polling. The OTel Collector uses 50-100MB of RAM depending on pipeline complexity.

On a production VICIdial cluster processing 200,000+ daily calls, the total overhead of the observability stack is less than what a single poorly-written AGI script adds per call. If you're worried about performance, profile your AGI scripts before cutting monitoring.

One exception: if you enable very high-cardinality labels (like including the full caller ID number as a metric label), Prometheus will eat memory proportionally. Keep labels to low-cardinality values — server name, channel type, queue name, disposition status. Not phone numbers, not channel IDs, not caller names.

---

## What We Covered

You now have:
- **Metrics** from `res_statsd` and AMI events flowing through OTel Collector to Prometheus
- **Dashboards** in Grafana for cluster health, SIP status, and per-server deep dives
- **Distributed traces** following individual calls through the Asterisk dialplan
- **Alerts** for trunk failures, channel exhaustion, registration storms, and queue backups
- **CDR-based analytics** exported as Prometheus metrics

The monitoring itself is the easy part. The hard part is building the habit of looking at dashboards before they page you, reviewing traces for calls that went wrong, and treating your observability stack as infrastructure that needs maintenance — not a one-time setup.

If you want to start smaller, skip the tracing. Get metrics into Prometheus, build the Cluster Overview dashboard, and set up the TrunkDown alert. That alone will save you from the next 3 AM "nobody noticed the trunk died" incident. Add tracing later when you need to debug specific call quality issues.

For more on the VICIdial monitoring side — real-time agent dashboards, campaign performance panels, and dialer efficiency metrics — see our [VICIdial Grafana real-time dashboard guide](https://vicistack.com/blog/vicidial-grafana-realtime-dashboard/). And if your Asterisk configuration needs attention before you start monitoring it, our [Asterisk configuration guide](https://vicistack.com/blog/vicidial-asterisk-configuration/) covers the SIP, codec, and NAT settings that affect call quality.

---

*Running a VICIdial cluster and want help setting up production observability? [Contact ViciStack](https://vicistack.com/contact/) — we've instrumented Asterisk environments from single-server shops to 10-server clusters processing a million calls a day.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/asterisk-otel-observability).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
