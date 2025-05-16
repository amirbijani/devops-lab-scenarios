# DevOps Training Scenario: Monitoring and Load Balancer Setup (With CPU & RAM Alerts)

## 📌 Objective

This training scenario will guide you (the DevOps trainee) to:

- Set up **Prometheus**, **Grafana**, and **Alertmanager**
- Deploy **Nginx** and configure it as a load balancer
- Monitor **Nginx**, **system CPU and RAM**
- Configure alerts and dashboards
- Perform functional and stress tests
- Deliver detailed documentation
- **[NEW] Extend to log monitoring, service uptime checks, and basic incident simulation**
- **[ENHANCED] Demonstrate the ability to investigate and interpret alerts with root cause analysis**
- **[ENHANCED] Apply monitoring insights to suggest potential infrastructure improvements or optimizations**

---

## 🗂 Project Structure

```
devops-training/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   ├── alert_rules.yml
├── alertmanager/
│   └── config.yml
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── datasource.yml
├── nginx/
│   ├── nginx.conf
│   └── html/
│       └── index.html
├── logs/
│   └── sample.log
├── blackbox/
│   └── config.yml
├── scripts/
│   └── simulate-incident.sh
```

---

## 🧬 Step-by-Step Tasks

### 1. Set Up Monitoring Stack

#### ✅ Task 1.1: Create Docker Compose

Include:
- `prometheus`
- `grafana`
- `alertmanager`
- `node_exporter`
- (Later) `nginx-prometheus-exporter`
- **[NEW] blackbox_exporter for uptime/HTTP checks**
- **[ENHANCED] Optional Loki and Promtail for log aggregation**

Expose:
- Prometheus: `:9090`
- Grafana: `:3000`
- Alertmanager: `:9093`
- Blackbox: `:9115`
- Loki: `:3100` (optional)

#### ✅ Task 1.2: Configure Prometheus

Add scrape configs for:

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'alertmanager'
    static_configs:
      - targets: ['alertmanager:9093']

  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-exporter:9113']

  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - http://nginx:80
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115
```

#### ✅ Task 1.3: Add Alerting Rules

File: `prometheus/alert_rules.yml`

```yaml
groups:
  - name: system.rules
    rules:
      - alert: HighCPUUsage
        expr: (100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage > 80% for 2 minutes"

      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "RAM usage > 85% for 2 minutes"

      - alert: NginxDown
        expr: up{job="nginx"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Nginx is down on {{ $labels.instance }}"

      - alert: HTTPProbeFailed
        expr: probe_success == 0
        for: 30s
        labels:
          severity: warning
        annotations:
          summary: "HTTP probe failed"
          description: "Blackbox probe to {{ $labels.instance }} failed"
```

Reference in `prometheus.yml`:

```yaml
rule_files:
  - "alert_rules.yml"
```

#### ✅ Task 1.4: Configure Alertmanager

Add `alertmanager/config.yml` with Slack/email/webhook (can be dummy initially). Optionally include route grouping, inhibition rules, and escalation timings.

---

### 2. Deploy Nginx Load Balancer

#### ✅ Task 2.1: Run Two Nginx Servers

```bash
docker run -d --name web1 -p 8081:80 nginx
docker run -d --name web2 -p 8082:80 nginx
```

#### ✅ Task 2.2: Configure Nginx Load Balancer

```nginx
upstream backend {
    server host.docker.internal:8081;
    server host.docker.internal:8082;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
    }

    location /nginx_status {
        stub_status;
        allow all;
    }
}
```

#### ✅ Task 2.3: Add `nginx-prometheus-exporter`

```bash
docker run -d   --name nginx-exporter   -p 9113:9113   nginx/nginx-prometheus-exporter   -nginx.scrape-uri http://nginx:80/nginx_status
```

---

### 3. Grafana Dashboards

#### ✅ Task 3.1: Provision Prometheus

Use Grafana provisioning to load Prometheus as a data source.

#### ✅ Task 3.2: Create Dashboards

- **System Dashboard**:
  - CPU usage over time
  - RAM usage
- **Nginx Dashboard**:
  - Requests per second
  - Active connections
  - 4xx/5xx error rates
- **[NEW] Uptime Dashboard**:
  - Target availability (from blackbox_exporter)
  - HTTP probe latency
- **[ENHANCED] Incident Heatmap**:
  - Visual map of when alerts occurred across the week
  - Useful to spot patterns of failures

---

### 4. Test and Validate

#### ✅ Task 4.1: Load Balancer Test

```bash
ab -n 1000 -c 10 http://localhost:8080/
```

#### ✅ Task 4.2: CPU/RAM Stress Test

Install `stress`:

```bash
sudo apt install stress
stress --cpu 2 --vm 1 --vm-bytes 512M --timeout 120s
```

Expected:
- Grafana CPU/RAM charts spike
- Prometheus alerts fire
- Use this opportunity to analyze system response times and correlation between resource saturation and service degradation

#### ✅ Task 4.3: Alertmanager Test

Stop Nginx or exporters. Validate alerts show in:
- Prometheus UI
- Alertmanager UI
- Confirm alerts group correctly and silence functionality works as expected

#### ✅ Task 4.4: Simulate Log Alerts (Optional Future Step)

Add `log-generator.sh` that writes errors to `sample.log`. Integrate with a tool like `Loki` or prepare for future log stack.
- **[ENHANCED] Write grep-style queries to extract "ERROR" and simulate noisy log scenarios**
- Use Loki with Grafana to build alert conditions on log patterns

#### ✅ Task 4.5: Uptime Probe Failure

- Temporarily stop Nginx
- Confirm blackbox_exporter detects failure
- Review alert timeline and recovery

---

## 📦 Deliverables

You must submit:

1. **Config files**:
   - `docker-compose.yml`, `prometheus.yml`, `alert_rules.yml`, `alertmanager/config.yml`, `nginx.conf`, Grafana provisioning
2. **Screenshots**:
   - Grafana dashboards
   - Prometheus targets
   - Alertmanager active alerts
3. **Written Documentation**:
   - What you built
   - What you tested
   - Problems faced + solutions
   - **[NEW] Suggestions for Next Extensions (e.g., CI/CD, logging, service mesh)**
   - **[ENHANCED] Postmortem: Pick one incident (e.g., CPU spike or downtime), describe what triggered the alert, how it was detected, and how it could be mitigated/prevented in future**

---

## 🧠 Learning Outcomes

You will learn:

- Monitoring stack setup (Prometheus, Grafana, Alertmanager)
- Load balancing with Nginx
- System metrics collection (CPU/RAM)
- Real-world testing + alert validation
- HTTP/uptime probe simulation
- Documentation and debugging skills
- **[NEW] Planning for observability and incident simulation**
- **[ENHANCED] How to extract insight from dashboards to make architectural or capacity planning decisions**
- **[ENHANCED] Skills to write and analyze post-incident reports using metrics, logs, and alert timelines**
