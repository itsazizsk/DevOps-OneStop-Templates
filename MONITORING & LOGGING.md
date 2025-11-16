# 1. MONITORING & LOGGING
### 1.1 Prometheus – Sample Configuration

#### Prometheus is used to scrape metrics from Kubernetes nodes, pods, exporters, and custom apps.

#### 📌 File: prometheus.yml
```
global:
  scrape_interval: 15s   # Default scrape interval

scrape_configs:

  # Scrape Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Scrape node exporter on Linux servers
  - job_name: 'node_exporter'
    static_configs:
      - targets:
          - '10.0.0.10:9100'
          - '10.0.0.11:9100'

  # Scrape Kubernetes pods (example)
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        replacement: $1
```
### **Real-time usage**

* Monitor K8s pods automatically

* Collect Linux server metrics

* Export custom app metrics over `/metrics`

### 1.2 Grafana – Sample Dashboard JSON

#### Grafana imports dashboards using JSON.

#### 📌 File: node-exporter-dashboard.json
```
{
  "annotations": {
    "list": []
  },
  "panels": [
    {
      "type": "graph",
      "title": "CPU Usage",
      "targets": [
        {
          "expr": "100 - (avg by (instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
          "legendFormat": "{{instance}}"
        }
      ]
    },
    {
      "type": "graph",
      "title": "Memory Usage",
      "targets": [
        {
          "expr": "(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100",
          "legendFormat": "{{instance}}"
        }
      ]
    }
  ],
  "schemaVersion": 16,
  "title": "Node Exporter Dashboard",
  "version": 1
}
```
### **Real-time usage (DevOps & SRE)**

* Import JSON → ready-made dashboard

* Visualize CPU, RAM, Disk, Latency

* Attach alerts using alert rules

### 1.3 ELK Stack (Elasticsearch + Logstash + Kibana)
#### 1.3.1 Logstash Pipeline Example

#### 📌 File: logstash.conf
```
input {
  beats {
    port => 5044       # Filebeat sends logs here
  }
}

filter {
  grok {
    match => { "message" => "%{IPORHOST:client} %{WORD:action} %{INT:bytes}" }
  }
  date {
    match => [ "timestamp", "ISO8601" ]
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "app-logs-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```
### 1.3.2 Filebeat Config

#### 📌 File: filebeat.yml
```
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/*.log

output.logstash:
  hosts: ["localhost:5044"]
```
### **Real-time usage**

* Define parsing logic in Logstash

* Send logs from Filebeat to ELK

* Store in Elasticsearch and visualize in Kibana

* Build alerts for 500 errors, latency, login failures

### 1.4 AWS CloudWatch – Logs Setup
#### 1.4.1 CloudWatch Agent Config (Linux)

#### 📌 File: cloudwatch-agent.json
```
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "system-logs",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/myapp/app.log",
            "log_group_name": "app-logs",
            "log_stream_name": "app-{instance_id}"
          }
        ]
      }
    }
  }
}
```

#### Deploy using CLI:
```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:cloudwatch-agent.json \
  -s
```
### 1.4.2 CloudWatch Metric Alarm (CLI)
```
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-0abcd1234 \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:alert-topic
```
### **Real-time usage**

* Sends EC2 logs & custom app logs

* Alerts on CPU, memory, disk events

* Used heavily in autoscaling decisions

### 1.5 Azure Monitor – Example Setup
#### 1.5.1 Log Analytics Workspace – Query (KQL)
#### Get error logs from AppService
```
AppServiceConsoleLogs
| where TimeGenerated > ago(1h)
| where Level == "Error"
| project TimeGenerated, Message
```
### 1.5.2 Azure VM Diagnostic Settings

#### Enabling logs:
```
az monitor diagnostic-settings create \
  --name "vm-diag" \
  --resource "/subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/myvm" \
  --logs '[{"category": "Syslog","enabled": true}]' \
  --workspace "<workspaceId>"
```
### 1.5.3 Application Insights Telemetry
#### Example: Request Failure Query
```
requests
| where success == "False"
| summarize count() by url, resultCode
```
### 1.5.4 Metrics Alert Example
```
az monitor metrics alert create \
  --name HighCPU \
  --resource-group myRG \
  --scopes "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM" \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m
```
### **Real-time usage**

* Monitoring AppService, AKS, Functions

* SIEM logs using Log Analytics workspace

* Alert rules → Email/SMS/ITSM
----------------------------
# 2. complete, ready-to-run “Monitoring Pack
###






### Below is a complete, ready-to-run “Monitoring Pack” you can copy into a GitHub repo or a local folder. It includes:

* Prometheus (scrape config, alert rules)

* Alertmanager (config + routes)

* Grafana (docker-compose + provisioning + a ready dashboard JSON)

* Node Exporter example

* ELK (Elasticsearch + Logstash + Kibana) + Filebeat pipeline

* CloudWatch Agent config + CLI install snippet

* Azure Monitor: Log Analytics + sample workbook (JSON) + az commands

* Example alerting rules, dashboards, and recommended folder/repo structure

* README with step-by-step deploy and verification commands

#### Everything is copy-paste ready. Replace secrets / region / IDs before using in production.
----------------------
### Repo layout (suggested)
```
monitoring-pack/
├─ prometheus/
│  ├─ prometheus.yml
│  ├─ alert_rules.yml
│  └─ docker-compose.yml     # (prometheus + alertmanager + node-exporter)
├─ alertmanager/
│  └─ alertmanager.yml
├─ grafana/
│  ├─ docker-compose.yml
│  ├─ provisioning/
│  │  ├─ dashboards/
│  │  │  └─ node-exporter-dashboard.json
│  │  └─ datasources/
│  │     └─ datasource.yml
│  └─ README.md
├─ elk/
│  ├─ docker-compose.yml
│  ├─ logstash/
│  │  └─ pipeline.conf
│  └─ filebeat/
│     └─ filebeat.yml
├─ cloudwatch/
│  └─ cloudwatch-agent.json
├─ azure/
│  ├─ workbook.json
│  └─ az_commands.txt
└─ README.md
```
### 1) PROMETHEUS + ALERTMANAGER + NODE_EXPORTER
#### **File**: `prometheus/prometheus.yml`
```
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

rule_files:
  - /etc/prometheus/alert_rules.yml
```
#### **File**: `prometheus/alert_rules.yml`
```
groups:
  - name: instance.rules
    rules:
      - alert: InstanceHighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High CPU on instance {{ $labels.instance }}"
          description: "CPU usage > 80% for more than 2 minutes."

      - alert: InstanceDown
        expr: up == 0
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "Instance down {{ $labels.job }} {{ $labels.instance }}"
          description: "No scrape data for 3 minutes."

      - alert: NodeFilesystemAlmostFull
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Less than 15% disk space available."
```
#### **File**: `alertmanager/alertmanager.yml`
```
global:
  resolve_timeout: 5m

route:
  receiver: 'team-email'
  group_by: ['alertname', 'job']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h

receivers:
  - name: 'team-email'
    email_configs:
      - to: 'oncall@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'alertuser'
        auth_password: 'PLACEHOLDER_PASSWORD'
        require_tls: true

# Optional: Slack receiver example (use webhook url in secrets)
#  - name: 'slack'
#    slack_configs:
#      - api_url: 'https://hooks.slack.com/services/XXXXX/XXXXX/XXXXX'
#        channel: '#alerts'
```

#### Note: In production store Slack/webhook/SMTP credentials in secrets and mount as files or environment variables.

#### **File**: `prometheus/docker-compose.yml`

### (run Prometheus + Alertmanager + Node Exporter + cAdvisor)
```
version: '3.7'
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
    ports:
      - "9090:9090"
    command:
      - '--storage.tsdb.retention.time=7d'
    depends_on:
      - alertmanager

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    volumes:
      - ../alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    ports:
      - "9093:9093"

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    network_mode: bridge
    pid: "host"
    ports:
      - "9100:9100"
    command:
      - '--path.rootfs=/host'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
```
### 2) GRAFANA — provisioning + dashboard
#### **File**: `grafana/docker-compose.yml`
```
version: '3.7'
services:
  grafana:
    image: grafana/grafana:9.0.0
    container_name: grafana
    user: "472" # grafana UID
    volumes:
      - ./provisioning:/etc/grafana/provisioning
      - ./provisioning/dashboards:/var/lib/grafana/dashboards
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer
    depends_on:
      - prometheus
```
### **File**: `grafana/provisioning/datasources/datasource.yml`
```
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```
### **File**: `grafana/provisioning/dashboards/dashboard.yml`
```
apiVersion: 1
providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    options:
      path: /var/lib/grafana/dashboards
```
### **File**: `grafana/provisioning/dashboards/node-exporter-dashboard.json`

### (Include the earlier Node Exporter JSON; trimmed here — copy full JSON below)
```
{
  "title": "Node Exporter Dashboard",
  "panels": [
    {
      "type": "graph",
      "title": "CPU Usage",
      "targets": [
        {
          "expr": "100 - (avg by (instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
          "legendFormat": "{{instance}}"
        }
      ]
    },
    {
      "type": "graph",
      "title": "Memory Usage",
      "targets": [
        {
          "expr": "(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100",
          "legendFormat": "{{instance}}"
        }
      ]
    }
  ],
  "schemaVersion": 16
}
```

### When Grafana boots, it will auto-import the data source and dashboard.

### 3) ELK (Elasticsearch + Logstash + Kibana) + Filebeat (docker-compose)
#### **File**: `elk/docker-compose.yml`
```
version: '3.7'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.9
    container_name: es
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.9
    container_name: logstash
    volumes:
      - ./logstash/pipeline.conf:/usr/share/logstash/pipeline/logstash.conf:ro
    ports:
      - "5044:5044"

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.9
    container_name: kibana
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:7.17.9
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/log:/var/log:ro
    depends_on:
      - logstash

volumes:
  esdata:
```
#### **File**: `elk/logstash/pipeline.conf`
```
input {
  beats {
    port => 5044
  }
}

filter {
  if [fileset][module] == "nginx" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "app-logs-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```
#### **File**: `elk/filebeat/filebeat.yml`
```
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/*.log
      - /var/log/nginx/*.log

output.logstash:
  hosts: ["logstash:5044"]

setup.kibana:
  host: "kibana:5601"
```
### 4) AWS CLOUDWATCH AGENT (for EC2 / on-prem)
#### **File**: `cloudwatch/cloudwatch-agent.json`
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "system-logs",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/myapp/app.log",
            "log_group_name": "app-logs",
            "log_stream_name": "app-{instance_id}"
          }
        ]
      }
    }
  }
}

#### **Install & start agent (Linux EC2 example)**
```
# For Amazon Linux 2 / RHEL-style
sudo yum install -y amazon-cloudwatch-agent

# Fetch config and start (adjust path)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/path/to/cloudwatch-agent.json \
  -s
```
#### **Example CloudWatch alarm (AWS CLI)**
```
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:alerts-topic
```
### 5) AZURE MONITOR (Log Analytics + Workbook)
#### **File**: `azure/workbook.json`

#### (Example workbook showing last 1 hour errors from AppService and CPU on VMs)
```
{
  "$schema": "https://github.com/Microsoft/Application-Insights-Workbooks/blob/master/schema.json",
  "items": [
    {
      "type": 4,
      "content": {
        "query": "AppServiceConsoleLogs | where TimeGenerated > ago(1h) | where Level == 'Error' | project TimeGenerated, Message",
        "title": "AppService Errors (last 1h)"
      }
    },
    {
      "type": 4,
      "content": {
        "query": "Perf | where TimeGenerated > ago(1h) and ObjectName == 'Processor' and CounterName == '% Processor Time' | summarize avg(CounterValue) by Computer",
        "title": "VM CPU usage (last 1h)"
      }
    }
  ]
}
```
### Example az commands to create workspace & diagnostic setting

#### **File**: `azure/az_commands.txt`
```
# create resource group
az group create -n monitoring-rg -l eastus

# create log analytics workspace
az monitor log-analytics workspace create -g monitoring-rg -n demo-law

# get workspace id
az monitor log-analytics workspace show -g monitoring-rg -n demo-law --query id -o tsv

# enable diagnostics on VM
az monitor diagnostic-settings create \
  --name "vm-diag" \
  --resource "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/myVM" \
  --workspace "/subscriptions/<sub>/resourceGroups/monitoring-rg/providers/Microsoft.OperationalInsights/workspaces/demo-law" \
  --logs '[{"category": "Syslog","enabled": true}]'
```
### 6) ALERTING & RUNBOOK EXAMPLES
### Prometheus Alert → Alertmanager flow

* Prometheus evaluates rules (`alert_rules.yml`) → fires → sends to Alertmanager (`alertmanager:9093`) → Alertmanager routes to receiver (email / Slack / webhook)

### Example runbook snippet for InstanceHighCPU alert
```
1. Alert received via Slack/email with instance label.
2. SSH into instance: ssh ec2-user@<instance-ip>
3. Check top/htop for process causing CPU load.
4. If process is an expected task (backup, cron), update alert threshold or schedule.
5. If rogue, restart service:
   sudo systemctl restart myapp
6. Confirm CPU drops below threshold.
7. Add root cause to ticketing system and close alert.
```
### 7) VERIFY & HEALTH CHECKS (commands)

* Prometheus UI: `http://<host>:9090/` → Status → Targets (should show UP)

* Alertmanager UI: `http://<host>:9093/`

* Grafana: `http://<host>:3000/` (user: `admin` / password `admin` if used above)

* Kibana: `http://<host>:5601/`

* Elasticsearch: `http://<host>:9200/` (curl returns cluster info)

### 8) BEST PRACTICES & PRODUCTION NOTES

* Security: never expose management ports publicly. Use VPN or private network / NAT. Protect Grafana/Kibana with authentication (LDAP/OAuth).

* Secrets: store sensitive keys in secret stores (AWS Secrets Manager, Azure Key Vault) and inject into containers via orchestration (K8s Secrets).

*Scaling: Elasticsearch requires cluster sizing; use multiple nodes and persistent volumes.

*Retention: configure Prometheus TSDB retention, Elastic index lifecycle management (ILM) and CloudWatch log retention policy.

*Alerting: avoid noisy alerts — tune for durations and thresholds. Use dedupe/grouping in Alertmanager.

* High availability: run multiple Prometheus instances with remote_write to long-term storage (Cortex/Thanos) for HA.

* Monitoring as code: keep configs in Git, use CI to validate before applying (lint Prometheus, kie commit rules).

### 9) README (quick start)

#### File: `README.md` (root)
```
# Monitoring Pack - quick start

## Prerequisites
- Docker & docker-compose installed
- Optional: AWS CLI / az CLI configured for cloud bits

## Start Prometheus + Grafana + Alertmanager (local)
cd prometheus
docker-compose up -d

cd ../grafana
docker-compose up -d

# Start ELK
cd ../elk
docker-compose up -d

# Verify services
open http://localhost:9090 (Prometheus)
open http://localhost:3000 (Grafana)
open http://localhost:5601 (Kibana)
```
### 10) Next steps I can do for you (pick one)

* A) Create this repo in your GitHub (I will produce a ready-to-paste folder content and a single downloadable ZIP).

* B) Generate step-by-step K8s manifests & Helm charts to run Prometheus + Grafana + Alertmanager + ELK inside a cluster (with PersistentVolumes, Services, Ingress rules).

* C) Add sample Slack + PagerDuty alert templates and secure secret injection examples.

* D) Produce a short runbook template per alert with escalation steps, CI/CD validations, and demo playbook.

#### Tell me which one you want next (A / B / C / D) or if you want all of them.
