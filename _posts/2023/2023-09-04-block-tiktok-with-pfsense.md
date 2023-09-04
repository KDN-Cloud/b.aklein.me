---
title: Block TikTok with pfSense
date: 2023-08-27
description: Block TikTok with pfSense
tags:
- tiktok
- pfsense
- pfblocker
---

The <a href="https://www.netify.ai/resources/applications/tiktok" target="_blank">following page</a> provides details on domains, platforms, networks and IPs used by TikTok.

The <a href="https://coygeek.com/docs/pfsense-tiktok/" target="_blank">coygeek page</a> provides detailed steps to add the list of TikTok resources to a custom block list.

Netgate has a <a href="https://forum.netgate.com/topic/154408/firefox-users-and-doh/15" target="_blank">forum post</a> has some pretty good info about blocking Firefox DoH (DNS over HTTP). To me I would rather block DoH to decrease the attack surface.

**pfSense**

Block DNS over HTTPS nonsense. DoH (firefox uses DoH, so could still reach TikTok)

https://pfsense.local/pfblockerng/pfblockerng_category_edit.php?type=dnsbl&rowid=5

**DNS Groups**

https://pfsense.local/pfblockerng/pfblockerng_category.php?type=dnsbl
