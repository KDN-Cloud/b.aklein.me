---
title: Block TikTok with pfSense + pfBlockerNG
date: 2023-08-27
updated: 2026-06-01
description: Block TikTok on your network using pfSense and pfBlockerNG — DNSBL, IP blocklists, DoH blocking, and ASN-level coverage.
tags:
- tiktok
- bytedance
- pfsense
- pfsense-plus
- pfblockerng
- dnsbl
- dns-blocking
- doh
- dns-over-https
- dot
- dns-over-tls
- firewall
- network-security
- homelab
- self-hosted
- blocklist
- unbound
- ip-blocking
- privacy
- parental-controls
---

The old coygeek post that everyone referenced for this is long gone. Since I run pfBlockerNG on my Netgate 6100 and have a `Block_Tik_Tok` DNSBL group already in production, figured it was time to write this up properly — including the parts the original post skipped, like IP and ASN blocking.

## Why bother blocking it at the firewall?

App-level blocking is a joke. Browser extensions don't help on mobile. If you want TikTok off your network — whether for your kids, a corporate policy, or because you'd prefer ByteDance not hoover up everything on your LAN — you need to do it at the DNS and IP layer. This post covers both.

## Prerequisites

- pfSense (Plus or CE) with pfBlockerNG-devel installed
- Unbound configured as your DNS resolver (pfBlockerNG DNSBL hooks into Unbound)
- pfBlockerNG enabled and the DNSBL feature turned on

---

## Part 1: DNSBL — Block TikTok Domains

### Create a new DNSBL Group

Navigate to:

```
Firewall → pfBlockerNG → DNSBL → DNSBL Groups → + Add
```

Or directly: `https://pfsense.lan/pfblockerng/pfblockerng_category.php?type=dnsbl`

**Group settings:**

| Field | Value |
|---|---|
| Name | `Block_Tik_Tok` |
| Description | `Block TikTok and ByteDance infrastructure` |
| Action | `Unbound` |
| Update Frequency | `Once a day` |
| Logging/Blocking Mode | `DNSBL WebServer/VIP` |

### Option A: Use an auto-updating feed (recommended)

The best maintained TikTok domain blocklist right now is from [M4jx on GitHub](https://github.com/M4jx/TikTokBlockList). It's derived from live mobile traffic captures — not just guesswork — and is formatted for pfBlockerNG DNSBL.

Under **DNSBL Source Definitions**, add:

| Field | Value |
|---|---|
| Format | `Auto` |
| State | `ON` |
| Source | `https://raw.githubusercontent.com/M4jx/TikTokBlockList/main/hosts` |
| Header/Label | `TikTok` |

This keeps your block list current without having to manually chase down new ByteDance domains every time TikTok rotates infrastructure.

### Option B: Manual DNSBL Custom_List

If you'd rather maintain a static list (or supplement the feed), paste the following into the **DNSBL Custom_List** section at the bottom of the group edit page:

```
abtest-va-tiktok.byteoversea.com
isnssdk.com
lf1-ttcdn-tos.pstatp.com
muscdn.com
musemuse.cn
musical.ly
p1-tt-ipv6.byteimg.com
p1-tt.byteimg.com
p16-ad-sg.ibyteimg.com
p16-tiktok-sg.ibyteimg.com
p16-tiktok-sign-va-h2.ibyteimg.com
p16-tiktok-va-h2.ibyteimg.com
p16-tiktok-va.ibyteimg.com
p16-va-tiktok.ibyteimg.com
p26-tt.byteimg.com
p3-tt-ipv6.byteimg.com
p9-tt.byteimg.com
sf1-ttcdn-tos.pstatp.com
sf16-ttcdn-tos.ipstatp.com
sf6-ttcdn-tos.pstatp.com
sgsnssdk.com
tiktok.com
tiktokcdn-in.com
tiktokcdn.com
tiktokcdn.com.c.bytetcdn.com
tiktokcdn.com.c.worldfcdn.com
tiktokcdn.com.wsdvs.com
tiktokcdn.liveplay.myqcloud.com
tiktokv.com
ttlivecdn.com
ttlivecdn.com.c.worldfcdn.com
ttlivecdn.com.wsdvs.com
ttoversea.net
```

> **Note:** pfBlockerNG DNSBL does **not** support regex or wildcard entries in the Custom_List. One domain per line, no patterns.

---

## Part 2: IP Blocking — Because DNS Isn't Enough

DNS blocking alone can be bypassed. The TikTok app (especially on mobile) can use DoH or DoT to resolve domains through Cloudflare or Google, completely bypassing your local resolver. You need IP-level blocking to close that gap.

### IPv4

Navigate to `Firewall → pfBlockerNG → IP → IPv4 → + Add`

| Field | Value |
|---|---|
| Name | `TikTok_IPv4` |
| Description | `TikTok IPv4 ranges` |
| State | `ON` |
| Source | `https://raw.githubusercontent.com/M4jx/TikTokBlockList/main/ipv4s` |
| Header/Label | `TikTok` |
| Action | `Deny Both` |
| Update Frequency | `Once a day` |

### IPv6

Navigate to `Firewall → pfBlockerNG → IP → IPv6 → + Add` — same settings, different source:

```
https://raw.githubusercontent.com/M4jx/TikTokBlockList/main/ipv6s
```

---

## Part 3: Block DoH (DNS over HTTPS)

This is the one most guides skip. If you block domains at the DNS level but leave DoH wide open, anything using a hardcoded DoH resolver (Firefox does this by default, TikTok's CDN can too) will just route around you.

Block known DoH providers via DNSBL. I have a dedicated `DoH` group for this. pfBlockerNG ships with a DoH feed, or you can reference the Netgate forum thread that covers it:

[https://forum.netgate.com/topic/154408/firefox-users-and-doh/](https://forum.netgate.com/topic/154408/firefox-users-and-doh/)

You'll also want a firewall rule blocking outbound TCP/UDP 853 (DoT) to anything that isn't your own resolver, and consider blocking port 443 to known DoH IPs if you want to go further.

---

## Part 4: Force Update + Verify

After saving everything:

```
Firewall → pfBlockerNG → Update → Run (Force) → All
```

Then from a client on your LAN:

```bash
nslookup tiktok.com
```

You should get back the DNSBL VIP (something like `10.10.10.1` or whatever your DNSBL WebServer/VIP is configured to) instead of a real IP.

Navigating to `tiktok.com` in a browser should hit the pfBlockerNG block page. On mobile, the TikTok app will open but videos won't load.

---

## A note on domain list staleness

TikTok's infrastructure is not static. ByteDance rotates domains and CDN endpoints regularly. A static custom list will drift over time — I've seen the byteimg.com and ibyteimg.com subdomains change on me. The M4jx repo is updated from live traffic captures and is currently the best maintained source I've found for this. Pair it with the IP blocklists and you've got solid coverage without having to babysit it.

For reference on what TikTok's full network footprint looks like, [netify.ai's TikTok page](https://www.netify.ai/resources/applications/tiktok) has good detail on domains, IPs, and ASNs.

---

## Result

<img src="https://raw.githubusercontent.com/KDN-Cloud/b.aklein.me/main/img/tiktok-blocked.png" alt="tiktok blocked">

DNS blocked at the firewall, IP ranges dropped, DoH neutered. That's a dead app.

