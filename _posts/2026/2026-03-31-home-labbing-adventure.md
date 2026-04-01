---
layout: post
title: "KDN Lab: What I've Been Building"
date: 2026-03-31
description: First post in a while — a brief homelab update covering Proxmox, pfSense, UniFi, and WireGuard
tags:
  - homelab
  - proxmox
  - pfsense
  - unifi
  - wireguard
  - ansible
  - linux
---

It's been a while. A long while, actually — almost three years since the last post here. A lot has happened in that time, mostly in the form of a homelab that has grown well past "a few VMs" into something I spend a genuinely embarrassing number of evenings on. This is a quick overview. A longer write-up on specific pieces will follow.

## The Foundation

The core of the lab runs on **Proxmox VE**. Several VMs and LXC containers live there — a mix of Debian, Ubuntu, and a few purpose-built containers for specific services. I won't list every service, but the footprint has grown enough that I now have a full internal DNS, monitoring stack (Prometheus + Grafana + InfluxDB), SSO via Authentik, and a self-hosted Gitea instance mirroring to GitHub. Most services are only reachable from inside the network or over VPN — intentionally.

## Networking: pfSense + UniFi

Routing and firewall duties are handled by **pfSense Plus** on a Netgate 6100. It's been rock solid. Combined with a full **UniFi** setup — CloudKey, switches, APs — the network is properly segmented with VLANs for different device classes and trust levels. IoT stuff lives in its own corner, lab infrastructure in another, and the family devices somewhere in between.

## Cloudflare: DNS, Tunnels, and Workers

Most of my domains are managed through **Cloudflare DNS**, which makes handling multiple domains straightforward — DNS records, proxying, and security settings all in one place.

For one public-facing communication service, I use **cloudflared** to run a Cloudflare Tunnel. No open inbound ports on the firewall, no exposed public IP — the tunnel originates from inside the network and Cloudflare handles the public side. It's a clean way to selectively expose something without touching the broader network posture.

On the static site side, a few things run on **Cloudflare Workers** — this blog included, via GitHub Pages, but other lightweight static projects live entirely on Workers with no origin server needed. Fast, free at small scale, and zero maintenance overhead.

## WireGuard for the Family

One of the more satisfying things I've built is the **WireGuard VPN setup**. Peers are provisioned through pfSense, and the whole workflow — IP assignment, config generation, QR codes — is automated via an **Ansible playbook**. The `ops_automation` project has grown into a proper multi-role structure handling everything from WireGuard peer creation to dotfiles to Docker stack management.

Family members are on it. Everyone gets their own peer config, and the network-only-accessible services are actually usable for them without any friction. That part works really well.

## What's Running Internally

The short version: a lot. Photo management (Immich), music (Navidrome), media automation, books and audiobooks, docs, uptime monitoring, log aggregation, and a handful of other things. Everything behind Nginx Proxy Manager with proper TLS, and Authentik in front of anything that needs an auth wall.

None of it is particularly groundbreaking — these are all well-known open source tools — but the integration work and keeping it all running cleanly is where most of the time goes.

---

More detail on specific setups coming. There's plenty to write about.
