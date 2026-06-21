---
layout: post
#comments: false
promote_deployonfriday: true
title: "I Banned Myself From My Own Server and Had to Use the VPS Console to Get Back In"
date: 2026-06-20
author: Anthony Klein
description: "A CrowdSec war story: how I got locked out of my own VPS during parser testing, what the recovery looked like, and the DDNS-based trusted IP allowlist setup that makes sure it never happens again."
tags:
  - crowdsec
  - homelab
  - security
  - self-hosted
  - ids
  - ips
  - intrusion-detection
  - allowlist
  - whitelist
  - ansible
  - ddns
  - wireguard
  - vultr
  - vps
  - proxmox
  - linux
  - sysadmin
  - infrastructure
  - devops
  - sre
  - firewall
  - crowdsec-bouncer
  - crowdsec-lapi
  - fleet-security
  - homelab-security
  - kdn-lab
  - variablenix
  - ssh
  - openssh
  - iptables
  - nftables
  - homelab-automation
  - ansible-playbook
  - bash
  - testing-gone-wrong
  - war-story
  - infrastructure-as-code
---

Here is something nobody puts in their CrowdSec tutorials: you can absolutely ban yourself.

I was in the middle of validating my custom OpenSSH 9.8 parser on `nexus-node`, my public-facing Vultr VPS. The parser rewrites the `sshd-session` program field back to `sshd` so that `crowdsecurity/sshd-logs` can pick up events correctly on newer OpenSSH builds. To test it I was running `logger` commands to simulate brute force log lines, checking that GateKeeper was receiving and parsing them correctly.

The problem is that I was doing all of this from inside an active SSH session on that same server.

GateKeeper processed the simulated events, decided the source IP looked like a brute force attempt, and issued a ban. The firewall bouncer on `nexus-node` enforced it immediately via iptables. My active SSH connection stayed alive for another minute or so, then dropped. WireGuard went with it. I was locked out of the box entirely from the outside.

The only way back in was the Vultr web console.

Once I was in through the console I deleted the decision from GateKeeper:

```bash
# From inside the lab, SSH to GateKeeper
ssh 192.168.70.84   # your LAPI host

docker exec crowdsec cscli decisions list
docker exec crowdsec cscli decisions delete --ip <your-home-ip>
```

The bouncer on `nexus-node` synced the deletion from LAPI within about 30 seconds and the iptables rule dropped. SSH came back. Lesson learned the hard way.

---

## The Fix: Fleet-Wide Trusted IP Allowlist

The solution is a CrowdSec parser-stage whitelist that fires before scenarios evaluate anything. If the source IP is trusted infrastructure, the event never reaches LAPI in the first place.

Create the whitelist file on each agent node at `/etc/crowdsec/parsers/s02-enrich/user/trusted-ips-whitelist.yaml`:

```yaml
name: user/trusted-ips-whitelist
description: "Whitelist trusted infrastructure IPs"
filter: "1 == 1"
whitelist:
  reason: "Trusted lab infrastructure"
  ip:
    - "192.168.70.84"    # crowdsec LAPI host (GateKeeper)
  cidr:
    - "192.168.70.0/24"  # lab VLAN
    - "192.168.10.0/24"  # main VLAN
    - "10.69.0.0/24"     # WireGuard mesh
```

Use whatever your actual VLANs and infrastructure IPs are. The `filter: "1 == 1"` means it evaluates every event, checking the source against your trusted list before scenarios run.

This covers your static infrastructure. Your home IP is a different story.

---

## Handling a Rotating Residential IP with DDNS

Hardcoding a home IP in a whitelist is a bad idea unless your ISP gives you a static address. Mine doesn't. The IP rotates, and when it does, the hardcoded entry is wrong and you're exposed again.

The fix is to not hardcode it at all. I use DDNS for my home connection, with `pfsense.yourdomain.com` resolving to whatever my current residential IP is. In Ansible `group_vars`, I resolve the DDNS hostname at playbook run time:

```yaml
# group_vars/all/crowdsec.yml
crowdsec_trusted_home_ip: "{{ lookup('community.general.dig', 'pfsense.yourdomain.com') }}"
```

This gets resolved fresh every time the playbook runs, so the whitelist stays accurate automatically. If your ISP rotated your IP last Tuesday, the next time you run `ansible-playbook playbooks/crowdsec.yml` it picks up the new one.

The Ansible task that writes the whitelist file uses this variable:

```yaml
- name: Deploy trusted IP whitelist
  template:
    src: trusted-ips-whitelist.yaml.j2
    dest: "{{ crowdsec_config_dir }}/parsers/s02-enrich/user/trusted-ips-whitelist.yaml"
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: "0644"
  notify: restart crowdsec
```

And the Jinja template includes the home IP alongside the static entries:

```yaml
name: user/trusted-ips-whitelist
description: "Whitelist trusted infrastructure IPs"
filter: "1 == 1"
whitelist:
  reason: "Trusted lab infrastructure"
  ip:
    - "192.168.70.84"
    - "{{ crowdsec_trusted_home_ip }}"
  cidr:
    - "192.168.70.0/24"
    - "192.168.10.0/24"
    - "10.69.0.0/24"
```

Run it fleet-wide:

```bash
ansible-playbook -i hosts.ini playbooks/crowdsec.yml
```

Every agent gets the updated whitelist with the current home IP, and CrowdSec is restarted to pick it up.

---

## One More Thing: Test From a Different Machine

This sounds obvious in hindsight. If you are validating a parser that fires on SSH events, do not run your test commands from inside an SSH session on that same host. Log in through WireGuard from a lab machine first, or use the out-of-band console before you need to.

The parser works great now. And I have not banned myself since.

---

[ > aklein.me ]

