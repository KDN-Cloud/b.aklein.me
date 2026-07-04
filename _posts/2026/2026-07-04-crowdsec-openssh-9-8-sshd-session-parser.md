---
layout: post
#comments: false
promote_deployonfriday: true
title: "Writing a Custom CrowdSec Parser for OpenSSH 9.8's Process Rename"
date: 2026-07-04
author: Anthony Klein
description: "OpenSSH 9.8 quietly renamed sshd to sshd-session and broke CrowdSec's SSH brute force detection fleet-wide. Here's the custom s00-raw parser I wrote to fix it, why it works, and how I rolled it out with Ansible."
tags:
  - crowdsec
  - crowdsec-parser
  - crowdsec-collections
  - crowdsec-agent
  - crowdsec-lapi
  - crowdsec-hub-and-spoke
  - crowdsec-homelab
  - crowdsec-sshd
  - crowdsec-sshd-session
  - crowdsec-openssh
  - crowdsec-openssh-9.8
  - crowdsec-brute-force
  - crowdsec-detection
  - crowdsec-fix
  - crowdsec-custom-parser
  - crowdsec-s00-raw
  - crowdsec-statics
  - crowdsec-filter
  - crowdsec-fleet
  - crowdsec-ansible
  - crowdsec-linux
  - crowdsec-docker
  - crowdsec-proxmox
  - crowdsec-wireguard
  - crowdsec-whitelist
  - crowdsec-remediation
  - openssh
  - openssh-9.8
  - openssh-process-rename
  - openssh-sshd-session
  - openssh-compatibility
  - openssh-brute-force
  - openssh-linux
  - openssh-debian
  - openssh-ubuntu
  - sshd-session
  - sshd-session-rename
  - sshd-session-parser
  - sshd-session-crowdsec
  - sshd-session-fix
  - sshd-logs
  - sshd-brute-force
  - ssh
  - ssh-security
  - ssh-brute-force
  - ssh-brute-force-detection
  - ssh-brute-force-prevention
  - ssh-intrusion-detection
  - ssh-log-parsing
  - ssh-linux
  - ssh-hardening
  - parser
  - parser-yaml
  - log-parsing
  - log-forwarding
  - log-pipeline
  - s00-raw
  - s00-raw-parser
  - syslog
  - rsyslog
  - rsyslog-forwarding
  - rsyslog-auth
  - rsyslog-sshd
  - auth-log
  - authpriv
  - graylog
  - graylog-siem
  - siem
  - intrusion-detection
  - intrusion-prevention
  - ids
  - ips
  - brute-force-detection
  - brute-force-prevention
  - fleet-security
  - homelab-security
  - homelab
  - homelab-automation
  - homelab-infrastructure
  - homelab-linux
  - self-hosted
  - self-hosted-security
  - ansible
  - ansible-playbook
  - ansible-role
  - ansible-template
  - ansible-fleet
  - ansible-crowdsec
  - infrastructure-as-code
  - gitops
  - automation
  - linux
  - linux-security
  - linux-sysadmin
  - ubuntu
  - debian
  - sysadmin
  - sre
  - infrastructure
  - devops
  - security-engineering
  - security-automation
  - docker
  - docker-compose
  - proxmox
  - wireguard
  - vps
  - vultr
  - kdn-lab
  - variablenix
  - silent-failure
  - package-update
  - upgrade-compatibility
---

OpenSSH 9.8 shipped with a change that broke SSH brute force detection across my entire fleet without a single error message to tell me it happened.

No crash, no log warning, no CrowdSec alert saying "hey, something changed." Just silence where there used to be signal. Brute force attempts kept coming in, the fleet kept seeing them, and CrowdSec quietly stopped doing anything about them.

This post is about what changed, why it broke detection, and the custom `s00-raw` stage parser I wrote to fix it across the fleet.

I covered the `sshd-session` problem briefly in the [fleet-wide CrowdSec hub-and-spoke post](https://b.aklein.me/2026/06/16/crowdsec-hub-and-spoke-fleet-security.html). This is the deeper version.

{% include note.html content="**Update (July 2026):** The upstream `crowdsecurity/sshd-logs` parser (v3.1) now handles `sshd-session` natively alongside `sshd` in its filter. If you are running a current CrowdSec install with an up-to-date collection, you likely do not need the shim described here. That said, not everyone upgrades on a schedule, and plenty of homelab installs are running versions that predate the fix. If you hit this problem and your collection is behind, the approach below still works. The broader technique of a targeted `s00-raw` rewrite to fix a parser compatibility issue without forking the upstream collection is worth understanding regardless of version." %}

---

## What Changed in OpenSSH 9.8

Prior to 9.8, OpenSSH handled incoming connections with a single `sshd` process. Each connection was a forked child of that parent, and the process name stayed `sshd` throughout the session. That is the name that ends up in syslog. That is the name that ends up in `/var/log/auth.log`. That is the name CrowdSec's default SSH parser is built around.

OpenSSH 9.8 split the connection handling into a dedicated per-session process and renamed it `sshd-session`. The parent process that listens on port 22 is still called `sshd`. But once a connection is accepted, the per-session process carrying out that connection is now `sshd-session`. All of the authentication-related log lines that CrowdSec actually cares about come from that process.

So in your logs, the program field changed:

```
# Before OpenSSH 9.8
Jun 20 03:14:22 nexus-node sshd[12345]: Failed password for invalid user admin from 1.2.3.4 port 51234 ssh2

# After OpenSSH 9.8
Jun 20 03:14:22 nexus-node sshd-session[12345]: Failed password for invalid user admin from 1.2.3.4 port 51234 ssh2
```

One word different. Big operational consequence.

At the time, the `crowdsecurity/sshd` collection's parser filtered on `evt.Parsed.program == 'sshd'` exclusively. It never matched `sshd-session`. So failed auth attempts kept landing in the log, rsyslog kept forwarding them to GateKeeper, GateKeeper received them, and the parser simply did not match. No detection, no decision, no ban.

You would never know unless you were actively checking whether brute force detections were actually firing.

---

## Why a Custom Parser at the `s00-raw` Stage

CrowdSec parses log events in stages. The `s00-raw` stage is the earliest point in the pipeline, where raw syslog lines come in before any enrichment or scenario evaluation happens. It is the right place to fix structural issues with event data before anything downstream tries to use it.

The fix is straightforward: intercept any event where the program field is `sshd-session`, rewrite it to `sshd`, and pass it along to the next stage. The rest of the pipeline never knows the rename happened. The `crowdsecurity/sshd` collection sees `sshd` like it always did, and detection works normally.

The alternative would be forking the official `crowdsecurity/sshd-logs` parser and maintaining a modified version locally. That is a worse idea. You would have to keep it in sync with upstream changes to the collection, and it would break the moment CrowdSec updates the official parser logic. The `s00-raw` shim is small, self-contained, and does not touch anything else.

---

## The Parser

The file lives at:

```
/home/user/crowdsec/config/parsers/s00-raw/ak/sshd-session-rename.yaml
```

Content:

```yaml
filter: "evt.Parsed.program == 'sshd-session'"
onsuccess: next_stage
name: ak/sshd-session-rename
description: "Rename sshd-session to sshd for OpenSSH 9.8+ compatibility"
statics:
  - parsed: program
    value: sshd
```

Line by line:

`filter: "evt.Parsed.program == 'sshd-session'"` fires only on events where the program field is `sshd-session`. Everything else passes through untouched.

`onsuccess: next_stage` tells CrowdSec to move to the next stage immediately when the filter matches and the static is applied. This is important. Without it, the event could continue through the current stage and potentially hit other parsers in `s00-raw` unnecessarily.

`name: ak/sshd-session-rename` sets the namespace. The `ak` prefix is my custom parser namespace. In a shared or multi-user setup you would use something like `user/` or your own handle. The name has to be unique across your parser tree.

`statics` is where the rewrite happens. `parsed: program` targets the parsed program field. `value: sshd` sets it. After this runs, the event looks like it always came from `sshd` as far as every downstream parser is concerned.

That is the whole parser. Nine lines.

---

## Testing It Before Rolling Out

Before deploying anything fleet-wide, use `cscli explain` to confirm the parser is actually matching and transforming events the way you expect.

Pull a sample auth.log line with `sshd-session` in it, then run:

```bash
docker exec crowdsec cscli explain \
  --type syslog \
  --log "Jun 20 03:14:22 nexus-node sshd-session[12345]: Failed password for invalid user admin from 1.2.3.4 port 51234 ssh2" \
  --verbose
```

What you want to see in the output:

- your `ak/sshd-session-rename` parser showing as matched
- the `program` field in the parsed stage showing `sshd` rather than `sshd-session`
- the `crowdsecurity/sshd-logs` parser picking it up correctly in the subsequent stage

If `sshd-session` is still showing in the parsed output after your parser runs, the parser file is either in the wrong directory, not being loaded, or has a syntax error. Check the path first. It needs to be under `s00-raw/`, not `s01-parse/` or `s02-enrich/`.

You can also verify the parser is loaded at all:

```bash
docker exec crowdsec cscli parsers list | grep sshd-session
```

If it does not show up, CrowdSec did not find it. Confirm the full path is correct relative to your config directory.

---

## rsyslog Forwarding

One thing that tripped me up: on some hosts, rsyslog was not forwarding `sshd-session` log lines at all.

The typical rsyslog config for forwarding auth logs to a central syslog target is:

```
auth,authpriv.*  @@192.168.70.84:514
```

That forwards by facility. The `auth` and `authpriv` facilities cover standard authentication messages. In most cases that includes `sshd-session` events. But if you are running a tighter rsyslog config, or if the facility tagging on your distro does not include `sshd-session` entries under `auth`, you may need an explicit program-based rule alongside it:

```
auth,authpriv.*                         @@192.168.70.84:514
if $programname == 'sshd-session' then  @@192.168.70.84:514
```

The second rule catches any `sshd-session` events that slip through the facility filter and forwards them directly by program name. This matters in my setup because those same log lines are also flowing into Graylog for SIEM visibility, and I want the forwarding to be consistent regardless of how a particular distro or kernel version handles the auth facility tagging.

Worth checking per-host rather than assuming the facility rule covers everything.

---

## Fleet Rollout With Ansible

Once the parser works in isolation, rolling it out fleet-wide is an Ansible task. The parser file gets templated and deployed to each agent node under the correct path, and CrowdSec is restarted to pick it up.

Directory structure in the Ansible role:

```
roles/crowdsec/
  templates/
    parsers/
      s00-raw/
        ak/
          sshd-session-rename.yaml.j2
  tasks/
    parsers.yml
```

The template is identical to the parser file above. No dynamic values needed for this one since it is the same on every node.

The task:

```yaml
- name: Deploy sshd-session compatibility parser
  template:
    src: parsers/s00-raw/ak/sshd-session-rename.yaml.j2
    dest: "{{ crowdsec_config_dir }}/parsers/s00-raw/ak/sshd-session-rename.yaml"
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: "0644"
  notify: restart crowdsec
```

`crowdsec_config_dir` is a host variable that resolves to the actual config path per node, since not every box uses the same user. On most of my fleet it resolves to something like `/home/user/crowdsec/config`. On the Pi4 it is different. That variable handles it without me having to hardcode paths or write conditional logic in the task.

Deploy it:

```bash
ansible-playbook -i hosts.ini playbooks/crowdsec.yml --tags parsers
```

After the playbook runs and CrowdSec restarts on each node, run `cscli parsers list | grep sshd-session` on a few hosts to confirm the parser is present and loaded.

---

## How to Confirm Detection Is Working Again

After the parser is deployed, you want to verify that brute force detection is actually firing and not just silently passing events through.

The blunt approach: watch the alerts.

```bash
docker exec crowdsec cscli alerts list --since 1h
```

If you have a public-facing SSH endpoint, you should see some hits within a reasonable window. If the list stays empty for an unusually long time and you know the box is getting probes, that is a sign something is still not matching.

You can also force the issue with a controlled test. From a machine outside the trusted IP allowlist, send a few bad auth attempts against the target host. Then check whether CrowdSec registered the scenario and issued a decision:

```bash
docker exec crowdsec cscli decisions list
```

If your IP shows up, the pipeline is working end to end: log line to rsyslog to GateKeeper to parser to scenario to decision.

And then delete the test decision when you are done:

```bash
docker exec crowdsec cscli decisions delete --ip <your-test-ip>
```

---

## What Upstream Eventually Did

The CrowdSec team shipped a fix in `crowdsecurity/sshd-logs` v3.1. The parser filter now reads:

```yaml
filter: "evt.Parsed.program in ['sshd-session', 'sshd']"
```

Both program names match. Detection works on modern OpenSSH without any shim.

That is the right fix. It is also the fix that did not exist when I first hit this problem, which is why the shim exists in my config at all. If you are on a current collection version and setting up CrowdSec fresh, run `cscli collections upgrade crowdsecurity/sshd` and confirm the parser version before reaching for a custom workaround.

If you are on an older install, or if you hit a similar parser compatibility issue with a different service after a package update changes a program name, the `s00-raw` rewrite technique is still the cleanest way to handle it. A small targeted shim at the earliest stage, no forking, no drift from upstream, easy to remove when it is no longer needed.

---

The thing I keep coming back to with this one is how quiet the failure was. No alert, no error, no indication that anything was wrong. If I had not been actively checking whether detections were flowing after the OpenSSH upgrade, I might have gone weeks without noticing. Silent failures in a security layer are the worst kind.

The fix was nine lines of YAML. Upstream eventually agreed. Hopefully this saves someone the time it took me to figure out why the alert count dropped off after a routine package update, whether that is on a current install or one that has not seen an upgrade in a while.

