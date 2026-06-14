---
layout: post
promote_deployonfriday: true
title: "I Built a Homelab SIEM with Graylog on Proxmox: Here's Everything I Learned the Hard Way"
date: 2026-06-14
lastmod: 2026-06-14
author: Anthony Klein
description: "A full step-by-step guide to running Graylog 7.x + OpenSearch + MongoDB on a Proxmox VM with Docker Compose, NFS-backed storage, fleet-wide rsyslog forwarding via Ansible, GELF container logging, and a reverse proxy entry point. Every gotcha I hit, including source IP loss through Docker NAT, MongoDB NFS crash loops, and the Graylog 7.x Setup mode trap, documented so you don't have to debug them yourself."
tags:
  - graylog
  - siem
  - proxmox
  - homelab
  - opensearch
  - mongodb
  - docker
  - docker-compose
  - self-hosted
  - selfhosted
  - observability
  - logging
  - centralized-logging
  - log-management
  - rsyslog
  - syslog
  - gelf
  - ansible
  - infrastructure-automation
  - homelab-siem
  - homelab-monitoring
  - homelab-security
  - homelab-infrastructure
  - homelab-logging
  - homelab-observability
  - homelab-proxmox
  - homelab-docker
  - homelab-ansible
  - homelab-grafana
  - homelab-prometheus
  - kdn-lab
  - variablenix
  - nfs
  - nas
  - synology
  - synology-nas
  - nfs-mount
  - opensearch-docker
  - graylog-docker
  - graylog-7
  - graylog-opensearch
  - pfsense
  - pfsense-logging
  - pfsense-syslog
  - pfsense-synology
  - wireguard
  - wireguard-vpn
  - site-to-site-vpn
  - nginx-proxy-manager
  - npm-proxy
  - reverse-proxy
  - ssl
  - linux
  - ubuntu
  - ubuntu-2204
  - sre
  - devops
  - sysadmin
  - systems-administration
  - infrastructure
  - infrastructure-as-code
  - security
  - security-operations
  - log-aggregation
  - log-forwarding
  - fleet-logging
  - prometheus
  - grafana
  - docker-networking
  - docker-nat
  - docker-logging-driver
  - docker-gelf
  - vm
  - virtual-machine
  - proxmox-ve
  - proxmox-vm
  - open-source
  - open-source-siem
  - elk-alternative
  - opensearch-single-node
  - syslog-tcp
  - syslog-udp
  - source-ip
  - index-retention
  - vm-max-map-count
  - kernel-tuning
  - homelab-networking
  - vlan
  - network-monitoring
  - security-monitoring
---

> **What you're building:** A production-grade centralized log management and SIEM stack running Graylog 7.x + OpenSearch + MongoDB inside Docker on a Proxmox VM, with fleet-wide rsyslog forwarding, GELF container logging, NFS-backed storage, and a reverse proxy entry point. By the end, every node in your lab ships logs to one place, and you'll actually know what's happening across your infrastructure for the first time.

I ran this lab blind for longer than I'd like to admit. Back in 2012 I had a VPS running Splunk Enterprise with a couple forwarders. Centralized logging was just part of the setup. When that VPS got sunset, I never replaced it. For years it was just Grafana dashboards for metrics and a lot of SSH-ing into nodes one at a time playing detective when something broke. When I finally stood up Graylog, I had 13 nodes reporting in within a few hours. The first thing I saw was a pfSense firewall block storm that had apparently been running for who knows how long. That alone was worth the setup time.

This guide covers every gotcha I hit: the MongoDB NFS crash loop, the Docker NAT source IP problem, the Graylog 7.x input activation trap that makes logs silently disappear, and the Proxmox-specific rsyslog situation nobody documents. Skip to what you need or follow it top to bottom.

> **Before you copy anything:** I left my real lab patterns in here on purpose so the examples stay grounded. If you see `192.168.70.22`, that is my Graylog host. If you see `10.0.0.x`, `logs.lab.yourdomain.com`, or a sample NFS path, swap those for whatever matches your environment.

---

## Architecture Overview

<figure class="diagram-figure">
  <img
    src="/assets/images/graylog-architecture-overview.svg"
    alt="Architecture overview showing the Graylog VM, OpenSearch, MongoDB, reverse proxy, and pfSense to Synology log archive flow."
    class="diagram-image">
</figure>

**Why split storage?**
OpenSearch and Graylog journal data live on NFS (your NAS). These are high-volume, and you want them on a large array, not eating your VM disk. MongoDB holds only Graylog's *configuration* (dashboards, inputs, stream rules), not log data, so it stays on a local named volume. MongoDB has a documented incompatibility with NFS due to file locking. Don't fight it.

---

## Prerequisites

- Proxmox host with a spare VM slot
- Ubuntu 22.04 LTS ISO
- A NAS/NFS share (Synology, TrueNAS, QNAP, etc.), optional but recommended for >1M logs/day
- Docker + Docker Compose v2 installed on the VM
- Nginx Proxy Manager or similar reverse proxy
- A wildcard or dedicated SSL cert for your lab domain

---

## Step 1: Provision the Proxmox VM

In the Proxmox UI, create a new VM with these specs:

| Setting | Value |
|---|---|
| OS | Ubuntu 22.04 LTS |
| CPU | 4–8 vCPU |
| RAM | **16GB minimum** because OpenSearch is memory-hungry |
| Disk | 60–100GB (local SSD for OS + MongoDB) |
| Network | VirtIO NIC, assign a static IP |

Boot, install Ubuntu, set a static IP in `/etc/netplan/`, and install Docker:

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
```

---

## Step 2: Set vm.max_map_count (Required for OpenSearch)

OpenSearch will silently fail or crash without this. Set it permanently:

```bash
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Verify
cat /proc/sys/vm/max_map_count
# Should output: 262144
```

> **Production note:** This is a kernel-level setting OpenSearch requires for its memory-mapped files. On a production system this lives in your Ansible `sysctl` role and gets applied at build time. It is not something you want to discover is missing after the fact.

---

## Step 3: Mount NFS Storage (Optional but Recommended)

If you have a NAS, mount your Graylog share over NFS before the stack comes up. This offloads all log data off the VM disk entirely.

Install NFS client:

```bash
sudo apt install -y nfs-common
```

Create the mount point:

```bash
sudo mkdir -p /mnt/nas/graylog
```

Add to `/etc/fstab`. Adjust the NFS export path for your NAS:

```
# Graylog NFS share
10.0.0.x:/path/to/your/nfs/export  /mnt/nas/graylog  nfs  defaults,_netdev,rw,hard,intr,rsize=131072,wsize=131072,timeo=14  0  0
```

The server IP and export path here are placeholders. Use the actual NFS server and shared path from your own NAS.

Mount and pre-create subdirectories with correct ownership:

```bash
sudo mount -a

# OpenSearch runs as UID 1000
sudo mkdir -p /mnt/nas/graylog/{opensearch,journal,data}
sudo chown -R 1000:1000 /mnt/nas/graylog/

# Verify
df -h | grep graylog
ls -la /mnt/nas/graylog/
```

> **Why `_netdev`?** Tells systemd to wait for network availability before mounting. Without it, a reboot can race the network coming up and leave the NFS mount broken, which means OpenSearch won't start.

---

## Step 4: Generate Credentials

Graylog needs two secrets. Generate them before writing your `.env`:

```bash
# GRAYLOG_PASSWORD_SECRET: random 96-char secret
openssl rand -hex 48

# GRAYLOG_ROOT_PASSWORD_SHA2: SHA256 of your admin password
echo -n 'YourAdminPassword' | sha256sum | awk '{print $1}'
```

Keep both values handy for the next step.

---

## Step 5: Create the Docker Compose Stack

Create your stack directory:

```bash
mkdir -p /opt/stacks/graylog
cd /opt/stacks/graylog
```

### `.env`

```env
# Generate: openssl rand -hex 48
GRAYLOG_PASSWORD_SECRET=REPLACE_WITH_96_CHAR_SECRET

# Generate: echo -n 'yourpassword' | sha256sum | awk '{print $1}'
GRAYLOG_ROOT_PASSWORD_SHA2=REPLACE_WITH_SHA256_HASH

# Must be the publicly reachable URI, used in Graylog email links
GRAYLOG_HTTP_EXTERNAL_URI=https://logs.lab.yourdomain.com/

TZ=America/Los_Angeles
```

`GRAYLOG_HTTP_EXTERNAL_URI` needs to match the real URL you will open in the browser. If this is wrong, links and some UI behavior get weird fast.

### `docker-compose.yml`

```yaml
services:

  mongodb:
    image: mongo:7.0
    container_name: graylog-mongodb
    restart: unless-stopped
    networks:
      - graylog
    volumes:
      - mongodb-data:/data/db
      - mongodb-config:/data/configdb

  opensearch:
    image: opensearchproject/opensearch:2.15.0
    container_name: graylog-opensearch
    restart: unless-stopped
    environment:
      - cluster.name=graylog
      - node.name=opensearch
      - discovery.type=single-node
      - action.auto_create_index=false
      - OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g
      - DISABLE_INSTALL_DEMO_CONFIG=true
      - DISABLE_SECURITY_PLUGIN=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      # NFS mount, OpenSearch data lives here
      - /mnt/nas/graylog/opensearch:/usr/share/opensearch/data
    networks:
      - graylog

  graylog:
    image: graylog/graylog:7.0
    container_name: graylog
    restart: unless-stopped
    depends_on:
      - mongodb
      - opensearch
    env_file: .env
    environment:
      - GRAYLOG_ELASTICSEARCH_HOSTS=http://opensearch:9200
      - GRAYLOG_MONGODB_URI=mongodb://mongodb/graylog
    ports:
      - "9000:9000"       # Web UI
      - "9833:9833"       # Prometheus metrics
      - "5140:5140/tcp"   # Syslog TCP (preserves source IPs through Docker NAT)
      - "5140:5140/udp"   # Syslog UDP
      - "12201:12201/udp" # GELF UDP
    volumes:
      - /mnt/nas/graylog/journal:/usr/share/graylog/data/journal
      - /mnt/nas/graylog/data:/usr/share/graylog/data
    networks:
      - graylog

volumes:
  mongodb-data:
    driver: local
  mongodb-config:
    driver: local

networks:
  graylog:
    driver: bridge
```

> **Why MongoDB uses local named volumes and not NFS:** MongoDB's entrypoint tries to `chown` its data directories at startup. NFS blocks this due to root squashing, causing a crash loop. MongoDB only holds Graylog's config metadata, not log data, so local storage is fine and avoids the locking issue entirely.

> **Why TCP for syslog, not UDP?** When Docker NATs traffic through the bridge network, UDP packets lose their source IP. Every log entry looks like it came from the Docker host. Using TCP (`@@` in rsyslog) preserves the real originating source IP in Graylog. This one will drive you crazy if you miss it.

> **OpenSearch version pin:** Do not upgrade past `2.15.0`. Graylog 7.x does not support OpenSearch 2.16+. Pin it explicitly and make sure nothing auto-updates this image.

---

## Step 6: Start the Stack

```bash
cd /opt/stacks/graylog

# Start. OpenSearch takes 30–60 seconds before Graylog can connect
docker compose up -d

# Watch OpenSearch first
docker compose logs -f graylog-opensearch

# Once you see "started" in OpenSearch, watch Graylog
docker compose logs -f graylog

# You're ready when you see:
# Graylog server up and running.
```

Hit `http://<your-vm-ip>:9000`, then log in with `admin` and the password you hashed in Step 4.

---

## Step 7: Configure Graylog Inputs

Graylog doesn't collect *anything* until you configure Inputs. This is the most common "where are my logs?" moment for first-timers. There is no default collection. It just sits there waiting.

Navigate to **System → Inputs → Launch new input**.

### Input 1: Syslog TCP (fleet-wide rsyslog)

| Field | Value |
|---|---|
| Type | Syslog TCP |
| Port | 5140 |
| Bind address | 0.0.0.0 |
| Name | Fleet Syslog TCP |

### Input 2: GELF UDP (Docker containers)

| Field | Value |
|---|---|
| Type | GELF UDP |
| Port | 12201 |
| Bind address | 0.0.0.0 |
| Name | Docker GELF |

> **Graylog 7.x gotcha:** New inputs start in "Setup mode" and won't receive logs until you explicitly activate them. After creating each input, go to **More actions → Exit Setup mode → Start input**. You'll see the green "Running" indicator when it's actually live. I spent longer than I'd like to admit wondering why nothing was showing up before I found this.

---

## Step 8: Fleet-Wide rsyslog Forwarding via Ansible

For a fleet of 10+ nodes, configure forwarding with Ansible rather than SSH-ing into each host individually.

On most of my hosts, this is all the file needs to be. Nothing fancy. Just point the node at the Graylog box and ship everything over TCP.

### Generic host config: `/etc/rsyslog.d/99-graylog.conf`

```conf
*.* @@192.168.70.22:5140;RSYSLOG_SyslogProtocol23Format
```

In my lab, `192.168.70.22` is the host where the Graylog stack is running in Docker. On another network, change that IP to wherever your Graylog syslog TCP input is listening.

That one line is enough for the majority of Ubuntu, Debian, and similar Linux hosts.

If you are following along in your own lab, this is the one value you are most likely to change in the whole post. Keep the port and format unless you intentionally built your input differently.

If you want the same file created by hand:

```bash
sudo tee /etc/rsyslog.d/99-graylog.conf > /dev/null << 'EOF'
*.* @@192.168.70.22:5140;RSYSLOG_SyslogProtocol23Format
EOF

sudo systemctl restart rsyslog
```

I stick with TCP here for a reason. It keeps the source IP intact when Graylog is behind Docker, which saves a lot of confusion later.

### Ansible playbook: `rsyslog-graylog.yml`

```yaml
---
- name: Configure rsyslog forwarding to Graylog
  hosts: standard_nodes   # exclude proxmox and pfsense, handled separately below
  become: true
  tasks:

    - name: Install rsyslog
      ansible.builtin.apt:
        name: rsyslog
        state: present
        update_cache: yes

    - name: Deploy Graylog forwarding config
      ansible.builtin.copy:
        dest: /etc/rsyslog.d/99-graylog.conf
        content: |
          *.* @@192.168.70.22:5140;RSYSLOG_SyslogProtocol23Format
        owner: root
        group: root
        mode: '0644'
      notify: Restart rsyslog

  handlers:
    - name: Restart rsyslog
      ansible.builtin.systemd:
        name: rsyslog
        state: restarted
        enabled: true
```

Run it:

```bash
ansible-playbook rsyslog-graylog.yml -i hosts.ini

# Test from any node
logger -n 192.168.70.22 -P 5140 -T --rfc5424 "test from $(hostname)"
```

Check Graylog's **Search** tab. The test message should appear within a few seconds.

---

## Step 9: Special Cases

### Proxmox Host

Proxmox uses `systemd-journald` only. There is no `rsyslog.service` running by default, and you cannot just `systemctl restart rsyslog`. Install it manually:

```bash
ssh root@your-proxmox-host

apt-get install -y rsyslog
cat > /etc/rsyslog.d/99-graylog.conf << 'EOF'
*.* @@192.168.70.22:5140;RSYSLOG_SyslogProtocol23Format
EOF
systemctl enable rsyslog && systemctl start rsyslog
```

You'll see Proxmox VM start/stop events, storage operations, cluster activity, and PVE daemon logs flowing in under `source: your-proxmox-hostname`.

### pfSense / OPNsense → Graylog

pfSense has a built-in syslog forwarder. No SSH required.

Same idea here: use the IP of your Graylog host, not mine.

Navigate to **Status → System Logs → Settings**:

| Field | Value |
|---|---|
| Enable Remote Logging | ✅ |
| Remote Log Servers | `192.168.70.22:5140` |
| Remote Syslog Contents | Everything (or select specific facilities) |

Save and apply. Firewall events, DHCP leases, and IDS alerts appear in Graylog immediately.

### pfSense → Synology NAS (Local Log Archive)

If you want pfSense logs written directly to disk on a Synology NAS, independent of Graylog, you can run rsyslog on the Synology and have pfSense forward there instead, or in addition.

**On the Synology NAS:**

Enable SSH in Control Panel → Terminal & SNMP, then:

```bash
ssh admin@your-synology-ip

# Create log directory
mkdir -p /volume1/logs/pfsense

# Create rsyslog config
sudo tee /etc/rsyslog.d/pfsense.conf << 'EOF'
# Receive syslog from pfSense on UDP 514
$ModLoad imudp
$UDPServerRun 514

# Write pfSense logs to dedicated file
if $fromhost-ip == '10.0.0.1' then /volume1/logs/pfsense/pfsense.log
& stop
EOF

sudo synoservicectl --restart rsyslogd
```

**On pfSense:**

Navigate to **Status → System Logs → Settings**:

| Field | Value |
|---|---|
| Enable Remote Logging | ✅ |
| Remote Log Servers | `your-synology-ip:514` |
| Remote Syslog Contents | Firewall Events, General System, DHCP |

> pfSense log visibility via its own UI is already solid. Firewall rules, DHCP leases, and IDS events are all browsable natively. The Synology archive is purely for retention and offline reference. If you later want to pull these into Graylog, you can add a second remote log server entry in pfSense pointing at your Graylog instance and run both simultaneously. No rsyslog changes are needed on the NAS.

### Docker Containers (GELF)

Add the logging driver to any compose service to forward container logs directly to Graylog:

```yaml
services:
  your-app:
    image: your-image
    logging:
      driver: gelf
      options:
        gelf-address: "udp://10.0.0.x:12201"
        tag: "your-app-name"
```

Requires the GELF UDP Input to be running (Step 7). Point `gelf-address` at the host where your Graylog stack is listening for GELF, not just the container name. Use meaningful tag values. They become searchable fields in Graylog and make filtering across containers much cleaner.

### Remote Nodes (VPS over WireGuard)

For VPS nodes with site-to-site WireGuard access back to your lab, rsyslog forwarding works identically. Just point at the Graylog VM's lab IP through the tunnel:

```bash
sudo tee /etc/rsyslog.d/99-graylog.conf > /dev/null << 'EOF'
*.* @@192.168.70.22:5140;RSYSLOG_SyslogProtocol23Format
EOF
sudo systemctl restart rsyslog
```

No special configuration needed as long as WireGuard is routing your lab subnet through the tunnel.

---

## Step 10: Reverse Proxy (Nginx Proxy Manager)

Expose the Graylog UI through NPM:

| Field | Value |
|---|---|
| Domain | `logs.lab.yourdomain.com` |
| Scheme | `http` |
| Forward Hostname/IP | your Graylog VM IP |
| Forward Port | `9000` |
| Websockets Support | ✅ |
| SSL Certificate | `*.lab.yourdomain.com` wildcard |
| Force SSL | ✅ |

> **Websockets required:** Graylog's UI uses websockets for live log streaming. Don't skip that toggle or the search stream view will silently not work.
>
> The domain, upstream IP, and cert here all need to match your own environment. If the browser URL and `GRAYLOG_HTTP_EXTERNAL_URI` disagree, expect strange behavior.

---

## Step 11: Index Retention Tuning

Out of the box, OpenSearch's default index set is ~4GB per index with no retention limit. Set this before your NAS fills up.

**System → Indices → Default index set → Edit**

| NAS allocation | Max indices |
|---|---|
| ~40GB | 10 |
| ~80GB | 20 |
| ~200GB | 50 |

Set **Rotation strategy** to `Index Size` at `4GB` and **Retention strategy** to `Delete` with your max index count. Graylog handles the rest automatically.

---

## Useful Commands

```bash
# Start the stack
cd /opt/stacks/graylog && docker compose up -d

# Stop the stack
docker compose down

# Restart only Graylog (e.g. after .env change)
docker compose up -d graylog

# Tail logs per container
docker compose logs -f graylog
docker compose logs -f graylog-opensearch
docker compose logs -f graylog-mongodb

# Check container status
docker compose ps

# Send a manual test syslog message
logger -n 10.0.0.x -P 5140 -T --rfc5424 "test from $(hostname)"

# Check vm.max_map_count (must be ≥ 262144)
cat /proc/sys/vm/max_map_count

# Check NFS mount health
df -h | grep graylog
```

---

## Fleet Coverage Reference

| Source | Method | Notes |
|---|---|---|
| Standard Linux nodes | rsyslog via Ansible | TCP, RFC5424 format |
| Proxmox host | rsyslog manual install | journald only by default, install rsyslog first |
| pfSense / OPNsense | Native syslog UI | Built-in forwarder, no packages needed |
| pfSense → Synology NAS | rsyslog on NAS, UDP 514 | Local disk archive, independent of Graylog |
| Docker containers | GELF logging driver | Per-service, tag everything |
| VPS nodes | rsyslog via WireGuard tunnel | Routes through site-to-site WG |

---

## Troubleshooting

**Graylog won't start / can't connect to OpenSearch**
OpenSearch takes 30–60 seconds to be ready. Watch `docker compose logs -f graylog-opensearch` and wait for the "started" line before expecting Graylog to come up. Patience.

**Logs not appearing after configuring rsyslog**
Check that the Input is in **Running** state, not Setup mode. In Graylog 7.x, new inputs require an explicit "Exit Setup mode → Start input" step. This is not obvious and not well documented.

**MongoDB crash loop on startup**
You've pointed MongoDB at an NFS mount. Move it to a local named volume. MongoDB's entrypoint `chown` is blocked by NFS root squashing. See the compose file in Step 5.

**rsyslog not found on Proxmox**
Proxmox ships without rsyslog. It uses journald only. `apt-get install rsyslog` and then enable it. Don't bother looking for `rsyslog.service` before installing it.

**AppArmor blocking rsyslog journal access**
If rsyslog starts but produces no output on a specific host, AppArmor may be blocking `/var/log/journal` access. Check `journalctl -xe | grep apparmor`. Fix is either an AppArmor exception for rsyslogd or disabling it on that host if it's not a security boundary you care about.

**Source IPs all showing as the Docker host IP**
You're using UDP syslog (`@` in rsyslog). Switch to TCP (`@@`). UDP loses source IPs through Docker's bridge NAT, so every log entry will look like it came from the same host. TCP fixes this.

---

## Production Notes

This stack is sized for a lab with 10–15 nodes. For a production deployment:

- **Multi-node OpenSearch cluster** for redundancy and index parallelism
- **Graylog Operations or Security tier** for built-in alerting, threat intel feeds, and compliance dashboards
- **mTLS on syslog inputs:** plaintext TCP is fine on an isolated lab VLAN, not fine for anything crossing untrusted networks
- **Dedicated journal disk** separate from OpenSearch data: high ingestion rates cause I/O contention on shared storage
- **Back up MongoDB:** it's small, but losing it means rebuilding every dashboard, input, and stream rule from scratch

---

## What's Next

With centralized logging running, the natural extensions are:

- **Streams:** route logs by source or facility into separate views (firewall, auth, container logs)
- **Pipelines:** enrich fields in real time (GeoIP on source IPs, severity normalization)
- **Alerts:** fire on failed SSH attempts, service crashes, or custom patterns
- **Grafana integration:** Graylog exposes Prometheus metrics on `:9833`; scrape it and overlay log volume on top of your existing dashboards

---

Running this in your lab? Hit me on the socials or drop a comment. If something in this guide is wrong or outdated for your version of Graylog, let me know. I'd rather fix it than leave someone debugging in circles.
