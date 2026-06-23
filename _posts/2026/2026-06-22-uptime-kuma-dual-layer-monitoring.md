---
layout: post
#comments: false
promote_deployonfriday: true
title: "Two-Layer Uptime Monitoring with Uptime Kuma: Internal Pi4 and External VPS"
date: 2026-06-22
author: Anthony Klein
description: "How I run two Uptime Kuma instances to cover blind spots that a single monitor can't see — a Pi4 on the Lab VLAN for internal service checks, and a Vultr VPS over a WireGuard split tunnel for external perspective and internet connectivity monitoring."
tags:
  - uptime-kuma
  - monitoring
  - homelab
  - self-hosted
  - raspberry-pi
  - raspberry-pi-4
  - pi4
  - vultr
  - vps
  - wireguard
  - split-tunnel
  - uptime
  - observability
  - infrastructure
  - devops
  - sre
  - docker
  - docker-compose
  - nginx-proxy-manager
  - npm
  - proxmox
  - linux
  - sysadmin
  - homelab-monitoring
  - internal-monitoring
  - external-monitoring
  - heartbeat
  - push-monitor
  - cron
  - bash
  - kdn-lab
  - variablenix
  - homelab-infrastructure
  - selfhosted
  - home-network
  - networking
  - cloudflare
  - letsencrypt
  - ssl
  - internet-monitoring
  - uptime-monitoring
  - service-monitoring
  - homelab-observability
  - status-page
---

A single Uptime Kuma instance has a fundamental blind spot: it can not tell you when the thing it's running on is the problem.

If your home internet drops, the monitor goes down with it. If the host it runs on has a resource issue, so does your alerting. You get silence when you most need a notification. For a homelab setup where you actually want to know when things are broken, that is not good enough.

The solution is two instances with different vantage points. One lives on the Lab VLAN and knows everything about your internal services. The other lives outside, has its own internet connection, and watches both your external presence and whether your home is reachable at all.

Here is how mine is set up.

---

## The Architecture

**Internal: Raspberry Pi 4 (on the Lab VLAN)**

The Pi4 runs Uptime Kuma in Docker and handles everything internal. It reaches services by private IP or internal hostname, which means it can monitor things that are not exposed to the internet at all. No reverse proxy needed for the checks themselves, low resource footprint, dedicated hardware so it is not competing with anything else on the fleet.

What it monitors:
- Internal services by LAN IP or `.lab.yourdomain.com` hostname
- Local device reachability (ping monitors for network equipment, NAS, etc.)
- Services that are only accessible from inside the VLAN

**External: Vultr VPS (nexus-node)**

The VPS runs its own Uptime Kuma instance. It has an independent internet connection, so if your home goes dark, this instance stays up and can alert you. It monitors your public-facing services, DNS, and critically, whether your home network is reachable.

It also monitors internal services over a WireGuard split tunnel, which I will cover below.

---

## Internal Instance Setup

The Pi4 runs Uptime Kuma in Docker Compose:

```yaml
services:
  uptimekuma:
    image: louislam/uptime-kuma:2
    container_name: Uptime-Kuma
    labels:
      - "crowdsec.enable=true"
      - "crowdsec.labels.type=uptime-kuma"
    hostname: uptimekuma
    mem_limit: 4g
    cpu_shares: 1024
    ports:
      - 3001:3001
    volumes:
      - /home/pi/uptime-kuma/data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      TZ: America/Los_Angeles
    restart: unless-stopped
```

A few things worth noting here. The `docker.sock` mount lets Uptime Kuma monitor Docker containers directly by name rather than just by port — useful if you want container-level health checks alongside service checks. The CrowdSec labels wire it into the fleet security mesh so any suspicious traffic hitting the Kuma instance gets picked up by the local agent.

Access it at `http://pi4-ip:3001` from the Lab VLAN, or put it behind a reverse proxy with a proper hostname if you want SSL locally. I proxy it through Nginx Proxy Manager with a Cloudflare DNS challenge cert so I get `kuma.lab.yourdomain.com` with a real certificate.

For internal monitoring, the checks are straightforward. Most services get an HTTP(S) monitor pointed at their internal URL:

```
https://grafana.lab.yourdomain.com
https://authentik.lab.yourdomain.com
http://192.168.70.10:631    # CUPS on the IoT VLAN
```

Ping monitors cover network hardware:

```
192.168.10.1   # router
192.168.10.2   # core switch
```

Nothing fancy here. The Pi4 stays on and awake, Docker keeps the container running, Uptime Kuma does its job.

---

## External Instance Setup

The VPS runs Uptime Kuma alongside Nginx Proxy Manager in a shared Docker network. This matters because Uptime Kuma does not expose its port directly — NPM handles SSL termination and proxying, so everything external hits NPM first.

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    networks:
      - proxy
    volumes:
      - ./data:/app/data
    # no ports: block — not exposed directly

  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    networks:
      - proxy
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - ./npm/data:/data
      - ./npm/letsencrypt:/etc/letsencrypt

networks:
  proxy:
    name: proxy
```

NPM proxies `status.yourdomain.com` to `uptime-kuma:3001` on the internal Docker network. Cloudflare DNS challenge handles the cert. The Uptime Kuma instance is only reachable through NPM.

What the external instance monitors:
- Public-facing services by their public domain names
- DNS resolution (TCP monitor on port 53 to your authoritative resolver)
- Your home network's WireGuard tunnel endpoint
- Internal services via split tunnel (see below)

---

## Monitoring Internal Services From the Outside via Split Tunnel

This is the part that makes the external instance actually useful for internal visibility.

The VPS is a WireGuard peer on the same mesh as the rest of the lab fleet. With a split tunnel configured correctly, traffic destined for your home VLANs routes through the tunnel while everything else goes out the VPS's own internet connection directly. This means the external Uptime Kuma can reach `192.168.70.x` addresses just like a machine sitting on your lab VLAN.

Why this matters: the external instance can detect failures that originate inside the lab but do not manifest as an internet outage. If a service goes down but your internet is fine, the external monitor catches it. If your internet goes down, the external monitor catches that too. You get coverage in both directions.

A WireGuard deep dive is a full post on its own (coming soon), but the key thing here is the `AllowedIPs` on the VPS peer config. A split tunnel means you list only the subnets you want routed through the tunnel, not `0.0.0.0/0`:

```ini
[Peer]
PublicKey = <pfSense WireGuard public key>
PresharedKey = <preshared key>
Endpoint = pfsense.yourdomain.com:51820
AllowedIPs = 192.168.10.0/24, 192.168.70.0/24, 10.69.0.0/24
PersistentKeepalive = 25
```

With this in place, the external Uptime Kuma can monitor internal services by their LAN IPs without sending all VPS internet traffic through your home connection. Your home internet going down means the tunnel drops, but the VPS remains independently operational and can alert you that the tunnel is gone.

---

## Heartbeat Monitor: Is My Home Internet Up?

The most useful monitor on the external instance is a push heartbeat that answers the question: is my home internet working?

The way push monitors work in Uptime Kuma: the external instance creates a monitor of type `Push`, which generates a unique URL. Something on your home network has to ping that URL on a regular schedule. If the pings stop, Uptime Kuma marks it down and alerts you. This inverts the normal check direction and lets you detect outbound internet failures from a home device's perspective.

On the Pi4 (or any always-on Lab VLAN device), a cron job sends the heartbeat:

```bash
# /usr/local/bin/heartbeat.sh
#!/usr/bin/env bash

PUSH_URL="https://status.yourdomain.com/api/push/<your-monitor-token>?status=up&msg=OK&ping="
PING_HOST="status.yourdomain.com"
LOGFILE="/var/log/heartbeat.log"

# Only send heartbeat if we can actually reach the outside
if ! ping -c 1 -W 2 "$PING_HOST" &>/dev/null; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') [SKIP] Cannot reach $PING_HOST" >> "$LOGFILE"
    exit 0
fi

LATENCY=$(ping -c 1 -W 2 "$PING_HOST" 2>/dev/null | awk -F'/' '/rtt/{print $5}')
LATENCY=${LATENCY:-0}

curl -sf --max-time 5 --retry 2 --retry-delay 1 \
    "${PUSH_URL}${LATENCY}" >> "$LOGFILE" 2>&1

echo "$(date '+%Y-%m-%d %H:%M:%S') [OK] Heartbeat sent (${LATENCY}ms)" >> "$LOGFILE"
```

Make it executable and add a logrotate config:

```bash
chmod +x /usr/local/bin/heartbeat.sh
```

```
# /etc/logrotate.d/heartbeat
/var/log/heartbeat.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
}
```

Uptime Kuma push monitors fire on a configurable interval. I have it set to every 60 seconds with a 120-second heartbeat window, and I run two cron entries to approximate 30-second cadence:

```
* * * * * /usr/local/bin/heartbeat.sh
* * * * * sleep 30 && /usr/local/bin/heartbeat.sh
```

One thing to watch: if you ever manually test this script with `sudo`, it creates `/tmp/heartbeat.lock` as root if you are using `flock`. The cron job runs as your normal user and cannot acquire the lock. Remove it:

```bash
sudo rm /tmp/heartbeat.lock
```

Then run the script as your cron user to confirm clean behavior:

```bash
sudo -u <your-user> /usr/local/bin/heartbeat.sh
```

---

## What Each Instance Owns

To keep this from getting confusing as you add monitors, keep the responsibilities clearly separated:

| Check Type | Internal (Pi4) | External (VPS) |
|---|---|---|
| Internal services by Lab VLAN IP | Yes | Via WireGuard tunnel |
| Public-facing services | No | Yes |
| Network hardware ping | Yes | No |
| Internet connectivity | No | Via push heartbeat |
| Tunnel health | No | Yes (monitor the WireGuard endpoint) |

The overlap on internal services is intentional. If a service goes down and only the internal monitor fires, the issue is probably the service itself. If both fire, the issue might be network-level. If neither fires but users report problems, something upstream of both monitors is wrong.

---

## Alerting

Both instances send notifications independently. I use Telegram for both so I get a message regardless of which monitor catches something first. Setup is the same on either instance: `Settings > Notifications > Add New > Telegram`, plug in your bot token and chat ID, and assign it to monitors.

The external instance also has a status page at `status.yourdomain.com` that shows public-facing service status. Uptime Kuma's built-in status page feature handles this under `Status Pages` in the sidebar. Add the monitors you want publicly visible, give it a title, and it auto-updates.

---

## The Result

When my home internet went down during a neighborhood outage last year, I got a Telegram notification from the external VPS within two minutes. The internal instance was obviously unreachable. But the VPS kept running, saw the heartbeat stop, and alerted me immediately.

That is the whole point of the dual-layer setup. One instance cannot do this alone.
