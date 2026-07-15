# ⚙️ SIEM Dashboard with IDS Integration - Setup Guide

This guide provides step-by-step instructions to recreate the SIEM lab environment using the Elastic Stack (Elasticsearch, Kibana, and Filebeat) integrated with the Snort Intrusion Detection System (IDS).

---## Table of Contents

- [Prerequisites](#-prerequisites)
- [Lab Environment](#-lab-environment)
- [Clone Repository](#step-1--clone-the-repository)
- [Install Elasticsearch](#step-3--install-elasticsearch)
- [Configure Elasticsearch](#step-4--configure-elasticsearch)
- [Install Kibana](#step-5--install-kibana)
- [Install Snort](#step-6--install-snort)
- [Configure Snort Rules](#step-7--configure-snort-rules)
- [Install Filebeat](#step-8--install-filebeat)
- [Configure Filebeat](#step-9--configure-filebeat)
- [Configure Target Server](#step-10--configure-the-target-server)
- [Start Snort](#step-11--start-snort)
- [Access Kibana](#step-12--access-kibana)
- [Attack Simulation](#step-13--simulate-attacks)
- [Verify Log Collection](#step-14--verify-log-collection)
- [Troubleshooting](#troubleshooting)


# 📋 Prerequisites

Before starting, ensure you have:

* Oracle VirtualBox installed
* Three virtual machines created
* Internet connectivity for package installation
* Ubuntu Server 22.04 (SIEM Server)
* Ubuntu Server 20.04 (Target Server)
* Kali Linux 2023.x (Attacker Machine)

---

# 🖥️ Lab Environment

| Machine       | Operating System | IP Address    | Purpose                                |
| ------------- | ---------------- | ------------- | -------------------------------------- |
| SIEM Server   | Ubuntu 22.04     | 192.168.56.10 | Elasticsearch, Kibana, Filebeat, Snort |
| Target Server | Ubuntu 20.04     | 192.168.56.20 | Apache & MySQL                         |
| Kali Linux    | Kali 2023.x      | 192.168.56.30 | Attack Simulation                      |

All systems should be connected using a **VirtualBox Host-Only Network (192.168.56.0/24)**.

---
# Step 1 – Clone the Repository
```bash
git clone https://github.com/yourusername/SIEM-Dashboard-IDS.git
cd SIEM-Dashboard-IDS
```
# Step 2 – Update Ubuntu
Run on both Ubuntu virtual machines.
```bash
sudo apt update
sudo apt upgrade -y
```
# Step 3 – Install Elasticsearch
```bash
sudo apt install elasticsearch -y
```
Enable the service.
```bash
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```
Verify Elasticsearch is running.
```bash
curl http://localhost:9200
```
# Step 4 – Configure Elasticsearch

For this isolated lab environment, disable Elastic Security.

> **Note:** This configuration is intended only for an isolated educational lab. Authentication and TLS should always be enabled in production deployments.

Edit the configuration file.
```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```
Add:
```yaml
xpack.security.enabled: false
```
Restart Elasticsearch.
```bash
sudo systemctl restart elasticsearch
```
# Step 5 – Install Kibana
```bash
sudo apt install kibana -y
```
Enable remote access.
```bash
sudo nano /etc/kibana/kibana.yml
```
Modify:
```yaml
server.host: "0.0.0.0"
```
Start Kibana.
```bash
sudo systemctl enable kibana
sudo systemctl start kibana
```
Verify service status.
```bash
sudo systemctl status kibana
```
# Step 6 – Install Snort

```bash
sudo apt install snort -y
```
Verify installation.
```bash
snort -V
```
# Step 7 – Configure Snort Rules

Open the local rules file.

```bash
sudo nano /etc/snort/rules/local.rules
```
Add the following rules.
```snort
alert tcp any any -> 192.168.56.20 any (msg:"NMAP SYN SCAN DETECTED"; flags:S; sid:1000001; rev:1;)
alert tcp any any -> 192.168.56.20 any (msg:"NMAP VERSION SCAN DETECTED"; flags:SA; sid:1000002; rev:1;)
alert tcp any any -> 192.168.56.20 any (msg:"NMAP AGGRESSIVE SCAN DETECTED"; flags:A; sid:1000003; rev:1;)
alert icmp any any -> 192.168.56.20 any (msg:"ICMP PING DETECTED"; itype:8; sid:1000004; rev:1;)
alert tcp any any -> 192.168.56.20 22 (msg:"SSH CONNECTION ATTEMPT"; sid:1000005; rev:1;)
alert tcp any any -> 192.168.56.20 80 (msg:"HTTP ACCESS DETECTED"; sid:1000006; rev:1;)
```

Ensure Snort loads the local rules.

Open:

```bash
sudo nano /etc/snort/snort.conf
```
Verify the following line exists.

```conf
include $RULE_PATH/local.rules
```
# Step 8 – Install Filebeat
```bash
sudo apt install filebeat -y
```
# Step 9 – Configure Filebeat

Open the configuration.

```bash
sudo nano /etc/filebeat/filebeat.yml
```
Configure Filebeat to monitor Snort alerts.
```yaml
filebeat.inputs:

- type: log
  enabled: true
  paths:
    - /var/log/snort/alert

output.elasticsearch:
  hosts: ["localhost:9200"]
```
Enable Filebeat.
```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
```
Verify status.
```bash
sudo systemctl status filebeat
```
# Step 10 – Configure the Target Server

Install Apache and MySQL.

```bash
sudo apt install apache2 mysql-server -y
```

Enable services.

```bash
sudo systemctl enable apache2

sudo systemctl enable mysql
```
Start services.

```bash
sudo systemctl start apache2

sudo systemctl start mysql
```
Verify Apache.
```bash
curl http://localhost
```
# Step 11 – Start Snort

Start Snort in IDS mode.

```bash
sudo snort \
-i enp0s8 \
-c /etc/snort/snort.conf \
-l /var/log/snort \
-A fast
```

> Replace **enp0s8** with the correct network interface if necessary.

Check available interfaces.

```bash
ip a
```
# Step 12 – Access Kibana
Open a browser and navigate to:

```text
http://192.168.56.10:5601
```

Create a Data View.

| Setting         | Value        |
| --------------- | ------------ |
| Name            | Snort Alerts |
| Index Pattern   | filebeat-*   |
| Timestamp Field | @timestamp   |

---
# Step 13 – Simulate Attacks
Run the following commands from the Kali Linux attacker machine.
## Host Discovery
```bash
nmap -sn 192.168.56.0/24
```
## SYN Scan
```bash
nmap -sS 192.168.56.20
```
## Version Detection

```bash
nmap -sV 192.168.56.20
```
## Aggressive Scan

```bash
nmap -A 192.168.56.20
```
## Web Enumeration

```bash
nikto -h http://192.168.56.20
```
## SSH Login Attempt

```bash
ssh wronguser@192.168.56.20
```

---

# Step 14 – Verify Log Collection

Check that Snort is generating alerts.

```bash
sudo tail -f /var/log/snort/alert
```
Verify Filebeat is running.

```bash
sudo systemctl status filebeat
```
Verify Elasticsearch.

```bash
curl http://localhost:9200
```
Open Kibana and confirm that new events appear in the dashboard after running attack simulations.
---

# Troubleshooting

## Elasticsearch Not Starting

Check service status.
```bash
sudo systemctl status elasticsearch
```
View logs.

```bash
sudo journalctl -u elasticsearch
```
## Kibana Not Accessible

Check service.
```bash
sudo systemctl status kibana
```
Verify the server host configuration.

```yaml
server.host: "0.0.0.0"
```
## Snort Produces No Alerts

Verify:

* Correct network interface
* Local rules are included
* Traffic is reaching the monitored interface
* Snort configuration contains:

```conf
include $RULE_PATH/local.rules
```
## Filebeat Not Shipping Logs

Check service.

```bash
sudo systemctl status filebeat
```
Test configuration.

```bash
sudo filebeat test config
```
Check output connectivity.

```bash
sudo filebeat test output
```
## Kibana Shows No Data

Verify:

* Elasticsearch is running.
* Filebeat is active.
* Snort is generating alerts.
* Data View uses `filebeat-*`.
* Time filter includes recent events.
---

# Expected Results

After completing the setup successfully, the environment should provide:

* Real-time Snort alert collection
* Automatic log shipping through Filebeat
* Elasticsearch indexing of IDS events
* Live Kibana dashboards
* Detection of Nmap scans
* Detection of ICMP traffic
* Detection of SSH connection attempts
* Detection of HTTP access events
---

# Lab Cleanup

Stop services if required.

```bash
sudo systemctl stop filebeat

sudo systemctl stop kibana

sudo systemctl stop elasticsearch

# Disclaimer

This guide is intended solely for educational use within an isolated laboratory environment. All testing should be performed only on systems you own or are explicitly authorised to assess. Never perform security testing against production or third-party systems without prior written permission.
