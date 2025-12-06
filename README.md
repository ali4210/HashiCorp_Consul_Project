# 🚀 HashiCorp Consul Multi-Node Cluster & Service Mesh

[![Consul Version](https://img.shields.io/badge/Consul-1.22.1-blueviolet)](https://www.consul.io/)
[![License](https://img.shields.io/badge/License-BUSL--1.1-blue)](https://github.com/hashicorp/consul/blob/main/LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-orange)](https://www.kernel.org/)

> A production-ready guide for building a multi-node HashiCorp Consul cluster with heterogeneous Linux environments for service discovery, health checking, and distributed configuration management.

## 📖 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Prerequisites](#-prerequisites)
- [Installation](#%EF%B8%8F-installation)
- [Network Configuration](#-network-configuration)
- [Cluster Setup](#-cluster-setup)
- [Essential Consul Commands](#-essential-consul-commands)
- [Service Discovery](#-service-discovery--health-checks)
- [Key/Value Store](#-keyvalue-store-operations)
- [Troubleshooting](#-troubleshooting-guide)
- [Production Best Practices](#-production-best-practices)

---

## 🎯 Overview

This project demonstrates a **fully operational HashiCorp Consul cluster** spanning multiple Linux distributions, showcasing enterprise-grade service mesh capabilities, distributed configuration management, and automated service discovery.

### What is Consul?

Consul is a distributed service mesh solution that provides:
- **Service Discovery**: Automatic detection and registration of services
- **Health Checking**: Monitor service and node health in real-time
- **Key/Value Store**: Centralized dynamic configuration management
- **Multi-Datacenter Support**: Connect services across multiple regions
- **Secure Service Communication**: Built-in encryption and authorization

---

## 🏗️ Architecture

### Cluster Topology

```
┌─────────────────────────────────────────────────────┐
│                  Kali-DC Datacenter                 │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────┐         ┌──────────────────────┐ │
│  │   MASTER    │◄────────┤   CLIENT NODES       │ │
│  │ (Kali Linux)│         │                      │ │
│  │   Server    │         │  • sensu-backend     │ │
│  │  (Leader)   │         │    (Ubuntu)          │ │
│  │             │         │                      │ │
│  │ 192.168.0.150│        │  • parrot            │ │
│  └─────────────┘         │    (Parrot OS)       │ │
│         │                │                      │ │
│         │                │  • sallu             │ │
│         │                │    (CentOS)          │ │
│         │                └──────────────────────┘ │
│         │                                          │
│         └──────────► Gossip Protocol (8301)       │
│                      HTTP API (8500)               │
│                      DNS Interface (8600)          │
└─────────────────────────────────────────────────────┘
```

### Node Information

| Node Name      | Role   | OS Distribution | IP Address    | Status | Port Bindings       |
|----------------|--------|-----------------|---------------|--------|---------------------|
| **master**     | Server | Kali Linux      | 192.168.0.150 | alive  | 8300, 8301, 8500, 8600 |
| **sensu-backend** | Client | Ubuntu       | 192.168.0.160 | alive  | 8301, 8500, 8600    |
| **parrot**     | Client | Parrot OS       | 192.168.0.183 | alive  | 8301, 8500, 8600    |
| **sallu**      | Client | CentOS          | 192.168.0.202 | alive  | 8301, 8500, 8600    |

**Datacenter**: `kali-dc`  
**Consul Version**: `1.22.1`  
**Cluster Type**: Single-Server with Multiple Clients

---

## 📋 Prerequisites

### System Requirements

- **Operating Systems**: Debian-based (Ubuntu, Kali, Parrot) or RHEL-based (CentOS, Rocky Linux)
- **RAM**: Minimum 512MB per node (1GB+ recommended)
- **Network**: All nodes must be on the same LAN or have direct connectivity
- **Firewall**: Ports 8300-8302, 8500, 8600 must be accessible

### Required Packages

Install these utilities on **all nodes**:

```bash
# For Debian/Ubuntu-based systems
sudo apt update && sudo apt install -y curl gnupg software-properties-common jq netcat

# For RHEL/CentOS-based systems
sudo yum install -y curl gnupg2 jq nc
```

---

## ⚙️ Installation

### Method 1: Official HashiCorp Repository (Recommended)

This method ensures you get the latest stable version directly from HashiCorp.

#### For Debian-Based Systems (Ubuntu, Kali, Parrot)

```bash
# Step 1: Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Step 2: Add HashiCorp repository
# IMPORTANT: Use 'bookworm' (Debian 12 base) to avoid 404 errors
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com bookworm main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

# Step 3: Update and install Consul
sudo apt update
sudo apt install -y consul

# Step 4: Verify installation
consul version
```

#### For RHEL-Based Systems (CentOS, Rocky Linux)

```bash
# Step 1: Add HashiCorp repository
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# Step 2: Install Consul
sudo yum install -y consul

# Step 3: Verify installation
consul version
```

### Method 2: Binary Installation (Universal)

```bash
# Download the latest Consul binary
CONSUL_VERSION="1.22.1"
wget https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip

# Extract and install
unzip consul_${CONSUL_VERSION}_linux_amd64.zip
sudo mv consul /usr/local/bin/
sudo chmod +x /usr/local/bin/consul

# Verify installation
consul version
```

---

## 🔥 Network Configuration

### Firewall Rules (UFW)

Open required ports on **ALL nodes**:

```bash
# Allow Consul Server RPC (Server only)
sudo ufw allow 8300/tcp comment 'Consul Server RPC'

# Allow Consul LAN Gossip (All nodes)
sudo ufw allow 8301/tcp comment 'Consul LAN Gossip TCP'
sudo ufw allow 8301/udp comment 'Consul LAN Gossip UDP'

# Allow WAN Gossip (Multi-datacenter setups)
sudo ufw allow 8302/tcp comment 'Consul WAN Gossip TCP'
sudo ufw allow 8302/udp comment 'Consul WAN Gossip UDP'

# Allow HTTP API and Web UI (All nodes)
sudo ufw allow 8500/tcp comment 'Consul HTTP API/UI'

# Allow DNS Interface (All nodes)
sudo ufw allow 8600/tcp comment 'Consul DNS TCP'
sudo ufw allow 8600/udp comment 'Consul DNS UDP'

# Reload firewall
sudo ufw reload
sudo ufw status numbered
```

### Firewall Rules (firewalld - RHEL/CentOS)

```bash
# Add Consul ports
sudo firewall-cmd --permanent --add-port=8300/tcp  # Server RPC
sudo firewall-cmd --permanent --add-port=8301/tcp  # LAN Gossip TCP
sudo firewall-cmd --permanent --add-port=8301/udp  # LAN Gossip UDP
sudo firewall-cmd --permanent --add-port=8500/tcp  # HTTP API
sudo firewall-cmd --permanent --add-port=8600/tcp  # DNS TCP
sudo firewall-cmd --permanent --add-port=8600/udp  # DNS UDP

# Reload firewall
sudo firewall-cmd --reload
```

### Port Reference Table

| Port  | Protocol | Purpose                          | Required On    |
|-------|----------|----------------------------------|----------------|
| 8300  | TCP      | Server RPC (Raft consensus)      | Server only    |
| 8301  | TCP/UDP  | LAN Gossip (Serf)                | All nodes      |
| 8302  | TCP/UDP  | WAN Gossip (Multi-DC)            | Servers only   |
| 8500  | TCP      | HTTP API & Web UI                | All nodes      |
| 8600  | TCP/UDP  | DNS Interface                    | All nodes      |

---

## 🔧 Cluster Setup

### Server Configuration (Kali Linux - Master)

Create or edit `/etc/consul.d/consul.hcl` on the **master** node:

```hcl
# Datacenter name - all nodes must match this
datacenter = "kali-dc"

# Data directory for persistent storage
data_dir = "/opt/consul"

# Bind to all network interfaces
client_addr = "0.0.0.0"
bind_addr = "192.168.0.150"      # Server's IP address
advertise_addr = "192.168.0.150"  # Address advertised to other nodes

# Server mode configuration
server = true
bootstrap_expect = 1              # Single-server setup (use 3 or 5 for HA)

# Enable Web UI
ui_config {
  enabled = true
}

# Enable DNS forwarding to public resolvers
recursors = ["8.8.8.8", "8.8.4.4"]

# Performance tuning (optional)
performance {
  raft_multiplier = 1             # Default: 1 (lower = faster, less stable)
}
```

### Client Configuration (Ubuntu, Parrot, CentOS)

Create or edit `/etc/consul.d/consul.hcl` on **each client** node:

```hcl
# CRITICAL: Datacenter must match the server
datacenter = "kali-dc"

# Data directory for agent state
data_dir = "/opt/consul"

# Bind to all interfaces for API/DNS access
client_addr = "0.0.0.0"

# THIS NODE'S CONFIGURATION
# Replace with the actual IP of THIS client node
bind_addr = "0.0.0.0"             # Listen on all IPv4 interfaces
advertise_addr = "192.168.0.XXX"  # THIS client's IP address

# Client mode (not a server)
server = false

# Join the cluster by connecting to the server
retry_join = ["192.168.0.150"]    # Master server IP
```

#### Example: sensu-backend (Ubuntu) Client

```hcl
datacenter = "kali-dc"
data_dir = "/opt/consul"
client_addr = "0.0.0.0"
bind_addr = "0.0.0.0"
advertise_addr = "192.168.0.160"  # sensu-backend's IP
server = false
retry_join = ["192.168.0.150"]
```

#### Example: parrot (Parrot OS) Client

```hcl
datacenter = "kali-dc"
data_dir = "/opt/consul"
client_addr = "0.0.0.0"
bind_addr = "0.0.0.0"
advertise_addr = "192.168.0.183"  # parrot's IP
server = false
retry_join = ["192.168.0.150"]
```

#### Example: sallu (CentOS) Client

```hcl
datacenter = "kali-dc"
data_dir = "/opt/consul"
client_addr = "0.0.0.0"
bind_addr = "0.0.0.0"
advertise_addr = "192.168.0.202"  # sallu's IP
server = false
retry_join = ["192.168.0.150"]
```

### Configuration Quick Reference

When configuring a new client, you only need to modify **ONE line**:

```hcl
advertise_addr = "YOUR_CLIENT_IP_HERE"
```

Everything else remains the same across all clients!

---

## 🚀 Starting the Cluster

### Step 1: Create Data Directory

On **all nodes**:

```bash
sudo mkdir -p /opt/consul
sudo chown consul:consul /opt/consul
```

### Step 2: Start Consul Service

On **all nodes**:

```bash
# Start Consul agent
sudo systemctl start consul

# Enable auto-start on boot
sudo systemctl enable consul

# Check service status
sudo systemctl status consul
```

### Step 3: Verify Cluster Formation

On **any node**:

```bash
# View cluster members
consul members

# Expected output:
# Node            Address             Status  Type    Build   Protocol  DC
# master          192.168.0.150:8301  alive   server  1.22.1  2         kali-dc
# sensu-backend   192.168.0.160:8301  alive   client  1.22.1  2         kali-dc
# parrot          192.168.0.183:8301  alive   client  1.22.1  2         kali-dc
# sallu           192.168.0.202:8301  alive   client  1.22.1  2         kali-dc
```

### Step 4: Access Web UI

Open your browser and navigate to:

```
http://192.168.0.150:8500/ui
```

---

## 💻 Essential Consul Commands

### Cluster Management

```bash
# View all cluster members
consul members

# View detailed member information
consul members -detailed

# Check agent configuration
consul info

# Reload configuration without restart
consul reload

# Gracefully leave the cluster
consul leave

# Force remove a failed node
consul force-leave <node-name>
```

### Agent Operations

```bash
# Check Consul agent status
consul agent -dev  # Development mode (single-node)

# Start agent in server mode
consul agent -server -ui -bootstrap-expect=1 -data-dir=/opt/consul

# Start agent in client mode
consul agent -data-dir=/opt/consul -retry-join=192.168.0.150

# View agent logs
sudo journalctl -u consul -f

# Validate configuration file
consul validate /etc/consul.d/consul.hcl
```

### Service Management

```bash
# List all registered services
consul catalog services

# View detailed service information
consul catalog nodes -service=<service-name>

# Register a service (via API)
curl -X PUT -d @service.json http://localhost:8500/v1/agent/service/register

# Deregister a service
consul services deregister <service-id>

# Check service health
consul catalog nodes -service=<service-name> -detailed
```

### Health Checks

```bash
# List all health checks
consul monitor -log-level=debug

# Check cluster health
curl http://localhost:8500/v1/health/state/any

# Check specific service health
curl http://localhost:8500/v1/health/service/<service-name>

# View node health
consul catalog nodes -near=_agent
```

### Key/Value Operations

```bash
# Put a key-value pair
consul kv put <key> <value>
consul kv put config/app/db_host "192.168.0.100"

# Get a value
consul kv get <key>
consul kv get config/app/db_host

# Get value in raw format (no metadata)
consul kv get -raw config/app/db_host

# List all keys with prefix
consul kv get -recurse config/

# Delete a key
consul kv delete <key>

# Delete all keys with prefix
consul kv delete -recurse config/app/

# Export KV data
consul kv export config/ > backup.json

# Import KV data
consul kv import @backup.json
```

### DNS Queries

```bash
# Query service via DNS
dig @127.0.0.1 -p 8600 <service-name>.service.consul

# Query service with datacenter
dig @127.0.0.1 -p 8600 <service-name>.service.kali-dc.consul

# Query node
dig @127.0.0.1 -p 8600 <node-name>.node.consul

# Query with SRV records (includes port)
dig @127.0.0.1 -p 8600 <service-name>.service.consul SRV
```

### Snapshot & Backup

```bash
# Create a snapshot (server only)
consul snapshot save backup.snap

# Restore from snapshot
consul snapshot restore backup.snap

# Inspect snapshot
consul snapshot inspect backup.snap
```

### ACL Management (Advanced)

```bash
# Bootstrap ACL system
consul acl bootstrap

# Create a new token
consul acl token create -description="My token" -policy-name=read-only

# List all tokens
consul acl token list

# Read token details
consul acl token read -id=<token-id>
```

### Event System

```bash
# Fire a custom event
consul event -name=deploy-app

# Fire event with payload
consul event -name=deploy-app -payload="v2.5.0"

# Fire event to specific nodes
consul event -name=restart -node=web-server-01

# Watch for events (see Watch section for detailed examples)
consul watch -type=event -name=deploy-app /path/to/handler.sh
```

### Watch Operations (Real-Time Automation)

```bash
# Watch a specific key for changes
consul watch -type=key -key=config/app/db_host /usr/local/bin/handler.sh

# Watch all keys with a prefix
consul watch -type=keyprefix -prefix=config/app/ /usr/local/bin/handler.sh

# Watch a service for instance changes
consul watch -type=service -service=web-app /usr/local/bin/update-lb.sh

# Watch service with only passing health checks
consul watch -type=service -service=web-app -passingonly=true /usr/local/bin/handler.sh

# Watch all services in catalog
consul watch -type=services /usr/local/bin/catalog-handler.sh

# Watch health checks in critical state
consul watch -type=checks -state=critical /usr/local/bin/alert.sh

# Watch for custom events
consul watch -type=event -name=deploy /usr/local/bin/deploy-handler.sh

# Watch nodes
consul watch -type=nodes /usr/local/bin/node-handler.sh
```

---

## 🔍 Service Discovery & Health Checks

### Registering a Service

Create a service definition file `/etc/consul.d/web-app.json`:

```json
{
  "service": {
    "name": "web-app",
    "port": 8080,
    "tags": ["production", "api", "v1"],
    "address": "192.168.0.160",
    "check": {
      "id": "web-app-http",
      "name": "HTTP API Health Check",
      "http": "http://localhost:8080/health",
      "method": "GET",
      "interval": "10s",
      "timeout": "5s"
    }
  }
}
```

Reload Consul to register the service:

```bash
sudo systemctl reload consul

# Or via API
curl -X PUT -d @/etc/consul.d/web-app.json http://localhost:8500/v1/agent/service/register
```

### Health Check Types

**HTTP Check**:
```json
"check": {
  "http": "http://localhost:8080/health",
  "interval": "10s",
  "timeout": "5s"
}
```

**TCP Check**:
```json
"check": {
  "tcp": "localhost:3306",
  "interval": "10s",
  "timeout": "3s"
}
```

**Script Check**:
```json
"check": {
  "args": ["/usr/local/bin/check-db.sh"],
  "interval": "30s",
  "timeout": "10s"
}
```

**TTL Check** (service reports its own health):
```json
"check": {
  "ttl": "30s",
  "deregister_critical_service_after": "90s"
}
```

### Querying Services

**Via DNS**:
```bash
# Standard query
dig @127.0.0.1 -p 8600 web-app.service.consul

# With datacenter
dig @127.0.0.1 -p 8600 web-app.service.kali-dc.consul

# Get only healthy instances
dig @127.0.0.1 -p 8600 web-app.service.consul SRV
```

**Via HTTP API**:
```bash
# Get all instances
curl http://localhost:8500/v1/catalog/service/web-app

# Get only healthy instances
curl http://localhost:8500/v1/health/service/web-app?passing=true
```

**Via Consul CLI**:
```bash
# List services
consul catalog services

# Get service details
consul catalog nodes -service=web-app
```

---

## 💾 Key/Value Store Operations

The Consul KV store is a distributed configuration database.

### Basic Operations

```bash
# Store a configuration value
consul kv put global/app/log_level "INFO"

# Store JSON configuration
consul kv put config/database/settings '{"host":"192.168.0.100","port":5432}'

# Retrieve a value
consul kv get global/app/log_level

# Get raw value (without metadata)
consul kv get -raw global/app/log_level

# List all keys under a prefix
consul kv get -recurse global/app/

# Delete a key
consul kv delete global/app/log_level

# Delete all keys under a prefix
consul kv delete -recurse global/app/
```

### Advanced: Watching for Changes (Automation)

Consul's **watch** feature enables real-time automation by executing handlers when data changes. This is critical for dynamic infrastructure management.

#### Watch Types

Consul supports watching different resources:

| Watch Type | Purpose | Use Case |
|------------|---------|----------|
| `key` | Monitor a specific KV key | Config changes, feature flags |
| `keyprefix` | Monitor all keys under a prefix | Application configuration groups |
| `services` | Monitor all services | Service catalog changes |
| `service` | Monitor a specific service | Load balancer updates, failover |
| `checks` | Monitor health check state | Alerting, automated remediation |
| `event` | Monitor custom events | Deployment notifications |

#### Example 1: Watch a Key (Configuration Change)

Create watch configuration `/etc/consul.d/watch-config.json`:

```json
{
  "watches": [
    {
      "type": "key",
      "key": "global/app/log_level",
      "handler_type": "script",
      "args": ["/usr/local/bin/reload-app.sh"]
    }
  ]
}
```

Handler script `/usr/local/bin/reload-app.sh`:

```bash
#!/bin/bash
# Reload application when log level changes

# Read the new value from stdin (Consul passes JSON data)
NEW_VALUE=$(jq -r '.[0].Value' | base64 -d)

echo "[$(date)] Log level changed to: $NEW_VALUE" >> /var/log/consul-watch.log

# Update application config
echo "LOG_LEVEL=$NEW_VALUE" > /etc/myapp/loglevel.conf

# Reload your application
systemctl reload my-app

# Send notification (optional)
curl -X POST https://slack.webhook.url -d "{\"text\":\"Log level updated to $NEW_VALUE\"}"
```

Make the script executable:

```bash
sudo chmod +x /usr/local/bin/reload-app.sh
```

#### Example 2: Watch a Service (Load Balancer Updates)

Monitor the `web-app` service and update HAProxy configuration:

```json
{
  "watches": [
    {
      "type": "service",
      "service": "web-app",
      "args": ["/usr/local/bin/update-haproxy.sh"]
    }
  ]
}
```

Handler script `/usr/local/bin/update-haproxy.sh`:

```bash
#!/bin/bash
# Update HAProxy backend when web-app instances change

# Parse service instances from Consul JSON
BACKENDS=$(jq -r '.[] | select(.Status=="passing") | "\(.Node.Address):\(.Service.Port)"' | tr '\n' ' ')

# Generate HAProxy config
cat > /etc/haproxy/backends.cfg <<EOF
backend web-app
    balance roundrobin
    $(for backend in $BACKENDS; do
        echo "    server srv-${backend} ${backend} check"
    done)
EOF

# Reload HAProxy
systemctl reload haproxy

echo "[$(date)] HAProxy updated with backends: $BACKENDS" >> /var/log/haproxy-watch.log
```

#### Example 3: Watch Health Checks (Automated Alerting)

Monitor critical service failures:

```json
{
  "watches": [
    {
      "type": "checks",
      "state": "critical",
      "args": ["/usr/local/bin/alert-critical.sh"]
    }
  ]
}
```

Handler script `/usr/local/bin/alert-critical.sh`:

```bash
#!/bin/bash
# Send alerts for critical health checks

# Parse failed checks
FAILED_CHECKS=$(jq -r '.[] | "\(.Node): \(.CheckID) - \(.Output)"')

# Send to monitoring system
while IFS= read -r check; do
    echo "[CRITICAL] $check" | mail -s "Consul Health Alert" ops@company.com
    
    # Also send to PagerDuty/Slack
    curl -X POST https://events.pagerduty.com/v2/enqueue \
      -H 'Content-Type: application/json' \
      -d "{\"routing_key\":\"$PAGERDUTY_KEY\",\"event_action\":\"trigger\",\"payload\":{\"summary\":\"$check\",\"severity\":\"critical\"}}"
done <<< "$FAILED_CHECKS"
```

#### Example 4: Watch Events (Deployment Notifications)

Monitor custom deployment events:

```json
{
  "watches": [
    {
      "type": "event",
      "name": "deploy",
      "args": ["/usr/local/bin/handle-deploy.sh"]
    }
  ]
}
```

Fire events manually:

```bash
# Trigger deployment event
consul event -name=deploy -payload="v2.5.0"
```

Handler script `/usr/local/bin/handle-deploy.sh`:

```bash
#!/bin/bash
# Handle deployment events

VERSION=$(jq -r '.[0].Payload' | base64 -d)

echo "[$(date)] Deployment started: version $VERSION"

# Run deployment tasks
cd /opt/myapp
git fetch
git checkout $VERSION
npm install
systemctl restart myapp

echo "[$(date)] Deployment completed: version $VERSION"
```

#### Running Watches via CLI

Instead of configuration files, run watches directly:

```bash
# Watch a key
consul watch -type=key -key=global/app/log_level /usr/local/bin/reload-app.sh

# Watch a service with passing checks only
consul watch -type=service -service=web-app -passingonly=true /usr/local/bin/update-lb.sh

# Watch all services
consul watch -type=services /usr/local/bin/catalog-changed.sh

# Watch health checks in critical state
consul watch -type=checks -state=critical /usr/local/bin/alert.sh
```

#### Debugging Watch Handlers

Test your handler manually:

```bash
# Simulate Consul watch data
echo '[{"Key":"global/app/log_level","Value":"SU5GTw=="}]' | /usr/local/bin/reload-app.sh

# Check handler logs
tail -f /var/log/consul-watch.log

# Monitor watch execution
consul monitor -log-level=debug | grep watch
```

### Consul Template Integration

Install Consul Template:

```bash
# Download and install
wget https://releases.hashicorp.com/consul-template/0.37.4/consul-template_0.37.4_linux_amd64.zip
unzip consul-template_0.37.4_linux_amd64.zip
sudo mv consul-template /usr/local/bin/
```

Create a template file `config.ctmpl`:

```hcl
# Application Configuration
{{ key "global/app/log_level" }}
{{ range service "database" }}
server {{ .Address }}:{{ .Port }}{{ end }}
```

Run Consul Template:

```bash
consul-template \
  -template "config.ctmpl:/etc/app/config.conf:systemctl reload app"
```

---

## 🛠️ Troubleshooting Guide

### Common Issues and Solutions

#### Issue 1: Nodes Not Joining Cluster

**Symptom**:
```
[ERROR] agent: failed to join: error="dial tcp 192.168.0.150:8301: connection refused"
```

**Solutions**:

1. **Check firewall**:
```bash
# Verify port 8301 is open
sudo ufw status | grep 8301
sudo netstat -tuln | grep 8301
```

2. **Verify server is running**:
```bash
# On server node
sudo systemctl status consul
consul members
```

3. **Check network connectivity**:
```bash
# From client node
ping 192.168.0.150
telnet 192.168.0.150 8301
```

4. **Verify datacenter mismatch**:
```bash
# All nodes must have the same datacenter
grep datacenter /etc/consul.d/consul.hcl
```

#### Issue 2: Datacenter Mismatch

**Symptom**:
```
[WARN] agent: member 'client-1' part of wrong datacenter 'dc1', expected 'kali-dc'
```

**Solution**:
```bash
# Edit config on client
sudo nano /etc/consul.d/consul.hcl

# Change:
datacenter = "dc1"
# To:
datacenter = "kali-dc"

# Restart
sudo systemctl restart consul
```

#### Issue 3: Service Check Failing

**Symptom**:
Service shows as "critical" in UI with error `dial tcp :8080: connect: connection refused`

**Solutions**:

1. **Verify service is running**:
```bash
sudo netstat -tuln | grep 8080
curl http://localhost:8080/health
```

2. **Check port conflicts**:
```bash
sudo lsof -i :8080
# Kill conflicting process if needed
sudo kill -9 <PID>
```

3. **Start mock service for testing**:
```bash
# Python HTTP server
python3 -m http.server 8080

# Or using netcat
while true; do echo -e "HTTP/1.1 200 OK\n\n" | nc -l -p 8080; done
```

#### Issue 4: DNS Resolution Fails

**Symptom**:
```
dig @127.0.0.1 -p 8600 web-app.service.consul
# Returns: connection refused
```

**Solutions**:

1. **Check DNS port binding**:
```bash
sudo netstat -tuln | grep 8600
```

2. **Add DNS recursors** (for external queries):
```hcl
# In /etc/consul.d/consul.hcl
recursors = ["8.8.8.8", "8.8.4.4"]
```

3. **Configure system DNS** (optional):
```bash
# Add to /etc/systemd/resolved.conf
[Resolve]
DNS=127.0.0.1:8600
Domains=~consul
```

#### Issue 5: Consul Service Won't Start

**Symptom**:
```
Job for consul.service failed because the control process exited with error code
```

**Solutions**:

1. **Check logs**:
```bash
sudo journalctl -u consul -n 50 --no-pager
```

2. **Validate configuration**:
```bash
consul validate /etc/consul.d/consul.hcl
```

3. **Check file permissions**:
```bash
sudo chown -R consul:consul /opt/consul
sudo chmod 755 /opt/consul
```

4. **Check systemd service file**:
```bash
# Verify ExecStart path
cat /etc/systemd/system/consul.service

# Should point to: /usr/bin/consul or /usr/local/bin/consul
which consul

# Update if needed
sudo systemctl daemon-reload
```

### Diagnostic Commands

```bash
# View detailed agent information
consul info

# Check Raft consensus status
consul operator raft list-peers

# Monitor live logs
sudo journalctl -u consul -f

# Debug mode logs
consul monitor -log-level=debug

# Check API health
curl http://localhost:8500/v1/agent/self

# Network connectivity test
consul rtt <node-name>
```

---

## ✅ Production Best Practices

### High Availability

1. **Use 3 or 5 servers** (not just 1):
```hcl
# On each server
bootstrap_expect = 3
```

2. **Distribute servers across availability zones**

3. **Enable automated backups**:
```bash
# Cron job for daily snapshots
0 2 * * * consul snapshot save /backup/consul-$(date +\%Y\%m\%d).snap
```

### Security Hardening

1. **Enable ACLs** (Access Control Lists):
```hcl
acl {
  enabled = true
  default_policy = "deny"
  enable_token_persistence = true
}
```

2. **Enable TLS encryption**:
```hcl
verify_incoming = true
verify_outgoing = true
verify_server_hostname = true
ca_file = "/etc/consul.d/ca.pem"
cert_file = "/etc/consul.d/server.pem"
key_file = "/etc/consul.d/server-key.pem"
```

3. **Enable gossip encryption**:
```bash
# Generate encryption key
consul keygen
# Output: qDOPBEr+/oUVeOFQOnVypxwDaHzLrD+lvjo5vCEBbZ0=
```

```hcl
# Add to all nodes
encrypt = "qDOPBEr+/oUVeOFQOnVypxwDaHzLrD+lvjo5vCEBbZ0="
```

### Monitoring & Observability

1. **Enable Prometheus metrics**:
```hcl
telemetry {
  prometheus_retention_time = "24h"
  disable_hostname = false
}
```

2. **Integrate with logging systems**:
```bash
# Forward logs to syslog
consul agent -syslog
```

3. **Set up alerting** for:
   - Node failures
   - Service health checks failing
   - Leader elections
   - Raft apply timeouts

### Performance Tuning

1. **Adjust Raft multiplier** (trade-off: speed vs stability):
```hcl
performance {
  raft_multiplier = 1  # 1 = fastest, 5 = most stable
}
```

2. **Tune gossip intervals** for large clusters:
```hcl
gossip_lan {
  gossip_interval = "200ms"
  probe_interval = "1s"
  probe_timeout = "500ms"
}
```

3. **Increase RPC hold timeout**:
```hcl
limits {
  rpc_rate = -1
  rpc_max_burst = 1000
  rpc_max_conns_per_client = 100
}
```

---

## 📚 Additional Resources

- **Official Documentation**: [https://developer.hashicorp.com/consul](https://developer.hashicorp.com/consul)
- **Consul Learn Tutorials**: [https://learn.hashicorp.com/consul](https://learn.hashicorp.com/consul)
- **GitHub Repository**: [https://github.com/hashicorp/consul](https://github.com/hashicorp/consul)
- **Community Forum**: [https://discuss.hashicorp.com/c/consul](https://discuss.hashicorp.com/c/consul)

---

## 📄 License

This project documentation is provided under the MIT License. Consul itself is licensed under the [Business Source License 1.1 (BUSL-1.1)](https://github.com/hashicorp/consul/blob/main/LICENSE).

---

## 🤝 Contributing

Contributions, issues, and feature requests are welcome! Feel free to check the issues page or submit a pull request.

---

## 👨‍💻 Author

**Saleem Ali**  
🔗 [LinkedIn](https://www.linkedin.com/in/saleem-ali-189719325/)  
🔗 [GitHub](https://github.com/ali4210)

---

## ⭐ Show Your Support

If this guide helped you set up your Consul cluster, please give it a ⭐️ on GitHub!

---

**Last Updated**: December 2025  
**Consul Version**: 1.22.1  
**Status**: Production Ready ✅
