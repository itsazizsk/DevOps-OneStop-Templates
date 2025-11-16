# 1. FILE HANDLING (Real-time DevOps Usage)

### Linux file handling is the MOST important skill for DevOps because logs, configs, YAML files, scripts — everything is file-based.

### 1.1 ls — List files
```
ls
ls -l
ls -lh
```
### Why used?

* View files, permissions, owners, size, timestamps

* First step before editing or applying a configuration

### Real-time example:

#### Check Kubernetes manifest files in ```/etc/kubernetes/manifests```.

### 1.2 cd — Change directory
```
cd /var/log
cd ~/project
```
### When used?

* Navigate logs, config directories while debugging.

### 1.3 cat — View files
```
cat app.log
```
### Real-time:

#### Check last 20 lines of log:
```
tail -n 20 app.log
```
### 1.4 tail, head — Log monitoring
```
tail -f /var/log/syslog
```

#### Used for real-time logs (DevOps troubleshooting).
---------------
### 1.5 cp, mv, rm
```
cp file.txt backup.txt
mv file.txt /tmp/
rm -rf folder/
```

### Used for:

* Moving scripts

* Deleting temporary files

* Rotating logs manually

### 1.6 touch & mkdir
```
touch deployment.yaml
mkdir /var/app
```

#### Used when preparing config directories.

### 1.7 find
```
find / -name "*.log"
```

### Used to locate:

* Config files

* Logs

* PID files

### 1.8 grep — Filter text
```
grep "Error" app.log
```

#### Real-time:
#### Filter only WARNING logs:
```
grep -i warning /var/log/messages
```
### 🔥 2. NETWORKING COMMANDS

#### DevOps = 70% troubleshooting networking problems.

### 2.1 ping
```
ping google.com
```

### Used to:

#### Test connectivity

#### DNS resolution

### 2.2 curl
```
curl http://localhost:8080/health
```

### Real-time:
#### Check microservice health endpoint.

### 2.3 wget
```
wget https://releases.hashicorp.com/terraform.zip
```

#### Used to download binaries (Terraform, kubectl, etc.)

### 2.4 netstat / ss
```
ss -tulnp
```

### Used to check:

* Which ports are listening

* Identify failed services

### Example:
#### Check if Nginx is listening on 80.

### 2.5 nslookup / dig
```
dig google.com
```

#### Used for DNS troubleshooting.

### 2.6 traceroute

#### Checks path packets travel.

### 2.7 ip
```
ip a
ip r
```

### Used to check:

* IP addresses

* Routes

* Network interfaces

### 🔥 3. PROCESS MANAGEMENT
### 3.1 ps
```
ps aux | grep nginx
```

### Why?
#### Find process ID for restarts or kills.

### 3.2 top / htop

#### Used to monitor:

* CPU

* Memory

* Load

* Running processes

###Real-time:
###Find which process is consuming 90% CPU.

### 3.3 kill
```
kill -9 <PID>
```

### Used when:

* Docker hangs

* App stuck

* Service not stopping

### 3.4 nohup
```
nohup python app.py &
```

### Run app in background.

### 🔥 4. PERMISSIONS & GROUPS
### 4.1 chmod
```
chmod 644 file
chmod +x script.sh
```

### Real-time:
### Give execute permission to shell script.

### 4.2 chown
```
chown nginx:nginx /var/www/
```

### Used to fix permission issues while deploying web apps.

### 4.3 Users & Groups

### Check users:
```
cat /etc/passwd
```

### Check groups:
```
cat /etc/group
```
### 4.4 sudo
```
sudo systemctl restart nginx
```

### DevOps requirement → run privileged commands safely.

### 🔥 5. SYSTEMCTL (SERVICE MANAGEMENT)

### Systemd controls all services.

### Common commands
```
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl status nginx
systemctl enable nginx
systemctl disable nginx
```
### **Real-time DevOps usage:**

#### ✔ Start/stop services like Docker, Kubelet
#### ✔ Check why Podman/node exporter not running
#### ✔ Auto-start services on reboot

#### Example:
```
systemctl status kubelet
```
### 🔥 6. SSH (MOST IMPORTANT FOR DEVOPS)
### 6.1 Basic SSH
```
ssh ubuntu@54.220.10.15
```

### Used to:

* Connect to EC2

* Login to AKS nodes

* Access jumpbox servers

### 6.2 SSH key generation
```
ssh-keygen -t rsa -b 4096
```

### Used for:

* GitHub auth

* VM access

* Ansible

* Secure automation

### 6.3 Copy SSH key
```
ssh-copy-id user@host
```

### Enables passwordless login.

### 6.4 SCP
```
scp file.txt ubuntu@server:/tmp/
```

### Used to upload:

* Terraform files

* Kubernetes YAML

* Logs

* Packages

### 6.5 SSH config file

`~/.ssh/config`
```
Host prod-server
    HostName 3.22.17.90
    User ubuntu
    IdentityFile ~/.ssh/id_rsa
```

### Now login with:
```
ssh prod-server
```
