# 1. SHELL & PYTHON SCRIPTING (DevOps-Focused)
### 1.1 Basic Shell Scripts (Bash)

#### Below are the must-know shell scripting concepts for DevOps + samples used in real-time pipelines.

### 1.1.1 Basic Script Structure
```
#!/bin/bash
echo "Hello from shell"
```

### **Real-time use**: health checks, file cleanup, automation inside Jenkins, ADO self-hosted agent scripts.

### 1.1.2 Variables
```
NAME="Aziz"
echo "Welcome $NAME"
```
### 1.1.3 Conditions (if-else)
```
#!/bin/bash
if [ -f /etc/passwd ]; then
  echo "File exists"
else
  echo "File not found"
fi
```

### Real-time use: checking config before deployment, validating tools.

### 1.1.4 Loops
```
for i in {1..5}; do
  echo "Iteration $i"
done
```
#### Real-time use: looping servers, logs, objects.

### 1.1.5 Functions

```
hello() {
  echo "Hello from function"
}
hello
```
### 1.1.6 Reading User Input
```
read -p "Enter username: " user
echo $user
```
### 1.1.7 Reading Files (Log Parsing)
```
while read line; do
  echo "Line: $line"
done < /var/log/syslog
```
### 1.1.8 Exit Codes (CI/CD Use)
```
if [ $? -ne 0 ]; then
  echo "Error found"
  exit 1
fi
```

### Real-time use: Fail CI pipeline if script fails.

### ✔ Real-Time Shell Script Example (AWS Deployment Check)
```
#!/bin/bash

# Check EC2 instance status
INSTANCE_ID=i-0abcd1234

STATUS=$(aws ec2 describe-instances \
  --instance-id $INSTANCE_ID \
  --query "Reservations[*].Instances[*].State.Name" \
  --output text)

echo "EC2 Status: $STATUS"

if [ "$STATUS" == "running" ]; then
  echo "Instance already running"
else
  aws ec2 start-instances --instance-ids $INSTANCE_ID
  echo "Instance started"
fi
```
### 1.2 AWK & sed Examples

### Most asked in interviews. Used for log parsing, automation, data extraction.

### 1.2.1 sed — Find and Replace
```
sed -i 's/old/new/g' file.txt
```

### Use case: updating configs, modifying YAML/JSON in pipelines.

### 1.2.2 sed — Delete Lines
```
sed -i '5d' file.txt        # delete line 5
sed -i '/error/d' log.txt   # delete lines containing "error"
```
### 1.2.3 awk — Print Columns
```
awk '{print $1, $3}' data.txt
```

### Use case: Filter logs, metrics, CSV.

### 1.2.4 awk — Conditional Filtering
```
awk '$3 > 80 {print $1, $2, $3}' cpu_usage.txt
```

### Real-time: fetch high CPU nodes.

### 1.2.5 awk — Summation
```
awk '{sum+=$2} END {print sum}' file.txt
```
### ✔ Real-Time Example: Get Pod Restarts > 5
```
kubectl get pods -A --no-headers | awk '$5 > 5 {print $1, $2, $5}'
```
### 1.3 Cron Jobs (Automation Scheduling)
### 1.3.1 Create Cron Job
```
crontab -e
```
### 1.3.2 Cron Syntax
```
* * * * * command
| | | | |
| | | | └── Day of week (0–6)
| | | └──── Month (1–12)
| | └────── Day of month (1–31)
| └──────── Hour (0–23)
└────────── Minute (0–59)
```
### 1.3.3 Example – Run Script Every 5 Minutes
```
*/5 * * * * /home/ubuntu/health_check.sh
```
### 1.3.4 Example – Backup S3 Daily
```
0 2 * * * aws s3 sync /data s3://mybucket/backup
```
### 1.4 Python Automation Scripts (DevOps-Focused)

### Use Python for:
* AWS automation
* Azure automation
* CI/CD scripting
* Kubernetes automation
* File processing
* Monitoring scripts

### 1.4.1 Python + AWS (boto3)
### List S3 Buckets
```
import boto3

s3 = boto3.client('s3')
for bucket in s3.list_buckets()['Buckets']:
    print(bucket['Name'])
```
#### Start/Stop EC2
```
import boto3

ec2 = boto3.client('ec2')
instance = 'i-0abcd1234'

ec2.start_instances(InstanceIds=[instance])
print("EC2 Started")
```
### 1.4.2 Python + Azure (azure-mgmt)

#### Install:
```
pip install azure-identity azure-mgmt-compute
```

#### List VMs:
```
from azure.identity import DefaultAzureCredential
from azure.mgmt.compute import ComputeManagementClient

cred = DefaultAzureCredential()
client = ComputeManagementClient(cred, "SUBSCRIPTION_ID")

for vm in client.virtual_machines.list_all():
    print(vm.name)
```
### 1.4.3 Python + Kubernetes (client-python)

#### Install:
```
pip install kubernetes
```

#### List pods:
```
from kubernetes import client, config
config.load_kube_config()

v1 = client.CoreV1Api()
pods = v1.list_pod_for_all_namespaces()

for p in pods.items:
    print(p.metadata.name, p.status.phase)
```
### 1.4.4 Python Script to Auto-Restart CrashLoop Pods
```
from kubernetes import client, config
config.load_kube_config()

v1 = client.CoreV1Api()
pods = v1.list_pod_for_all_namespaces()

for pod in pods.items:
    if pod.status.container_statuses:
        for s in pod.status.container_statuses:
            if s.restart_count > 5:
                print(f"Restarting {pod.metadata.name}")
                v1.delete_namespaced_pod(
                    name=pod.metadata.name,
                    namespace=pod.metadata.namespace
                )
```
### 1.4.5 Python File Automation (Regex + Log Parsing)

#### Extract errors from logs:
```
import re

with open("app.log") as f:
    for line in f:
        if re.search("ERROR", line):
            print(line.strip())
```
### 1.4.6 DevOps Script: Merge JSON/YAML Files
```
import yaml

with open('values1.yaml') as f1, open('values2.yaml') as f2:
    data1 = yaml.safe_load(f1)
    data2 = yaml.safe_load(f2)

merged = {**data1, **data2}

with open('final.yaml', 'w') as f:
    yaml.dump(merged, f)
```
### 1.4.7 CI/CD Helper Script — Check Version & Fail Build
```
import sys

version = open("VERSION").read().strip()
if not version.startswith("v"):
    print("Invalid version format. Must start with v")
    sys.exit(1)
```

#### Used in build pipelines before tagging.
