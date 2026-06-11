---
layout: post
title: "How I Think About Domain Architecture"
date: 2026-06-02
description: Most people buy a domain and call it done. Here's why I run four, what each one does, and how Domain Locker keeps it all from falling apart.
tags:
  - domains
  - infrastructure
  - homelab
  - self-hosted
  - dns
  - domain-management
  - cloudflare
  - information-architecture
  - personal-brand
  - domain-locker
  - personal-domains
---

Most people buy a domain, point it somewhere, and call it done. Maybe they add a `www` subdomain if they're feeling ambitious. That's fine. It works. It's also how you end up with `myname.com` doing twelve different jobs and none of them particularly well.

I took a different approach — not by grand design initially, but the logic solidified over time and now it's intentional enough to write down.

Among the domains I operate, four are directly tied to me personally — each with a distinct TLD, each doing a specific job:

**`aklein.pro`** is the professional face — the consulting business, the services page, the contact form. `.pro` signals intent without needing a tagline. Anyone landing there knows immediately this isn't a hobby site.

**`aklein.me`** is the personal layer. This blog lives at `b.aklein.me`. The `.me` TLD has a specific connotation — not a business, not a brand, just a person with something to say. It does that work before you read a single word.

**`aklein.studio`** is the creative side — home to Lounge24 Radio at [radio.aklein.studio](https://radio.aklein.studio/public/lounge24_radio). `.studio` fits because it's a production environment in the literal sense. I DJ and produce music, the station runs 24/7, and a studio is where that kind of work happens.

**`kdn.cloud`** is the infrastructure namespace. Internal lab services live here — `cv.lab.kdn.cloud`, `photos.lab.kdn.cloud`, `postings.lab.kdn.cloud`. The `.cloud` TLD contextualizes everything under it immediately. Nobody's going to stumble onto `kdn.cloud` expecting a personal blog.

The pattern is that each TLD carries meaning before the content does. You know what you're walking into before the page loads, hopefully. That's good information architecture — the URL itself is part of the UX.

## Keeping track of it all: Domain Locker

Four domains sounds manageable. Four domains with multiple subdomains each, spread across different registrars, with varying expiration dates and DNS configurations — that gets harder to keep in your head.

I self-host [Domain Locker](https://github.com/lissy93/domain-locker) internally, an open source portfolio tracker that gives me a single dashboard for everything: expiration dates, registrar info, DNS records, SSL status, and uptime. The kind of thing you don't think you need until a domain quietly expires because you forgot which email address the renewal notice goes to.

It runs in Docker, fits neatly into the existing lab stack, and takes about ten minutes to set up. If you're running more than two or three domains and you're not tracking them somewhere, you're one missed renewal away from an embarrassing outage. Domain Locker is the fix for that.
