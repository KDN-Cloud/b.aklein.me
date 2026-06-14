---
layout: post
title: "Goodbye Portainer, Hello Dockhand"
promote_deployonfriday: true
date: 2025-06-13
tags:
- docker
- docker-compose
- dockhand
- portainer
- homelab
- self-hosted
- infrastructure
- container-management
- compose-stacks
- docker-management
- wireguard
- site-to-site
- multi-environment
- remote-docker
- git-integration
- gitea
- svelte
- sveltekit
- linux
- proxmox
- authentik
- oidc
- sso
- open-source
- bsl
- privacy
- no-telemetry
- ops
- devops
- sysadmin
---

I ran Portainer for a long time. Longer than I should have, honestly. It was one of those things I kept around out of inertia more than anything else, because at some point the pain of dealing with it became background noise. You get used to it. You stop noticing the sluggishness. You chalk up the weird UI behavior to "just how it is." And then one day something tips the scale and you finally go looking.

That day came when Portainer ate my `docker-compose.yml`.

## The Portainer Experience, In Retrospect

Let me be clear about what I was using Portainer for: Docker Compose stack management. That's it. I have over 10 stacks spread across multiple environments connected over a WireGuard tunnel, and I needed a way to view, manage, and update them without having to SSH into boxes constantly. Simple enough ask, right?

The problem is Portainer is not a simple tool. It's a platform that grew and grew and grew. Every version added more surface area, more menus, more settings, more enterprise-flavored features that I have zero use for. The UI got heavier. Page loads got slower. And behind all that Chrome it had this maddening behavior where it would internalize your compose files. Pull them in, "manage" them, and then quietly become the source of truth in a way that felt backwards and opaque.

There's a specific kind of frustration that comes from opening your compose stack in Portainer, making what should be a trivial edit, and then not being entirely sure if what you're looking at is actually what's on disk anymore. The compose file is yours. It should live on disk. It should be readable by any tool at any time. When a management layer starts abstracting that away from you, you've crossed a line I don't want to be on the wrong side of.

And then there was the day it just... ate the file. Gone. I'm not going to get into the full post-mortem because it still irritates me, but long story short: I lost a compose file for a stack I'd been running for months, and Portainer's internal state was not a useful substitute. That was the moment the search started in earnest.

## What I Actually Need

My infrastructure isn't overly complicated but it's not trivial either. I'm running Proxmox with somewhere north of 10 VMs and LXCs, pfSense handling multi-VLAN routing, a site-to-site WireGuard tunnel connecting multiple physical locations, and a pile of Docker Compose stacks distributed across those environments. Things like Graylog, CrowdSec, Vaultwarden, Immich, Navidrome, Jellyfin, AzuraCast for Lounge24 Radio, NetBox, Authentik. The usual homelab chaos, but organized chaos.

What I needed from a Docker management tool was pretty narrow:

- Manage remote Docker hosts (not just local)
- Full control over Compose stacks without abstracting the compose files away from me
- Fast UI that doesn't make me feel like I'm waiting for a page to load circa 2012
- No telemetry nonsense, no "phone home," no cloud features baked into something I'm running on-prem
- Something I can actually trust

I evaluated a few options. Came close to just writing a custom wrapper around the Docker API and calling it a day. Then I found Dockhand.

## Enter Dockhand

[Dockhand](https://github.com/Finsys/dockhand) is a Docker management application built by Jarek Krochmalski. The tagline is "Docker management you will like" which sounds like marketing until you actually use it and realize they mean it.

It's fast. That was the first thing I noticed. Not "fast for a Docker UI" fast. Just fast. The frontend is built on SvelteKit and it shows. Pages snap. Actions feel instant. Coming from Portainer this was jarring in the best way.

The deployment is refreshingly simple:

```yaml
services:
  dockhand:
    image: fnsys/dockhand:latest
    container_name: dockhand
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data:/app/data
    environment:
      - ADMIN_USERNAME=${ADMIN_USERNAME}
      - ADMIN_PASSWORD=${ADMIN_PASSWORD}
      - TZ=America/Los_Angeles
```

That's it. No separate database container required by default, no multi-container setup to babysit on day one. Just up and running.

## Multi-Environment Support Done Right

This is the feature that sealed it for me. Dockhand has proper multi-environment support, meaning I can add my remote Docker hosts and manage stacks across all of them from a single interface. Given that I have environments separated by a WireGuard tunnel, this mattered a lot.

In Portainer, remote host management always felt like an afterthought bolted onto something designed for single-host use. Dockhand treats it as a first-class feature. You add your remote Docker hosts, they show up as distinct environments in the UI, and you can switch between them cleanly. No weirdness.

## Compose Stacks the Way They Should Work

This is the big one. Dockhand stores your compose files in `~/.dockhand/stacks/` on disk. When you edit a stack through the UI, you're editing the actual file. There's no intermediary format, no proprietary state machine, no "Dockhand's version" of your compose file floating around separately from the real one.

The visual YAML editor has syntax validation built in, handles environment variables properly, and doesn't corrupt your file on save. That last part sounds like it should be table stakes but here we are.

You also get three categories of stack management:

- **Native stacks**: Created and managed directly in Dockhand, compose file lives in the Dockhand data directory
- **Git stacks**: Deployed from a Git repo with webhook support and auto-sync on push. GitHub, GitLab, Gitea, and custom Git servers all work.
- **Detected stacks**: Existing Compose deployments running outside Dockhand that it picks up for read-only monitoring

That last one matters a lot during migration. You don't have to port everything over at once. Dockhand will find your existing stacks and let you watch them without taking ownership immediately.

## Git Integration That's Actually Useful

The Git stack functionality deserves its own callout. Being able to point Dockhand at a Gitea repo (which I self-host under the KDN-Cloud org) and have it deploy and sync from there is exactly how I want to manage stacks going forward. Set up a webhook, push a commit, done. No manual intervention.

The webhook endpoint is straightforward:

```
POST /api/git/webhooks/{repositoryId}
```

Works with whatever webhook format your Git host sends. I have this wired up to my internal Gitea instance and it just works.

## Everything Else

Dockhand also has real-time log streaming, an interactive terminal for shell access into containers, a file browser for getting in and out of container filesystems, and real-time stats on CPU/memory/network/disk I/O. The authentication supports SSO via OIDC (relevant since I run Authentik) plus local users.

The base OS layer is built from scratch using Wolfi packages via apko with every package explicitly declared in the Dockerfile. That kind of transparency matters when you're deciding what gets access to your Docker socket.

It's licensed under BSL 1.1, which means it's free for personal use, internal business use, non-profits, and education. It converts to Apache 2.0 in 2029. Not a concern for my use case.

## The Actual Switch

Migration was painless. I stood up Dockhand, pointed it at each of my Docker hosts, let it detect my existing stacks, and spent a day validating that everything looked right before I decommissioned Portainer. No downtime, no drama.

The stacks I cared most about I pulled into native management or wired up to Git repos. The ones I haven't touched I'm leaving as detected stacks for now. Gradual migration is possible and Dockhand doesn't force your hand.

## Honest Take

Portainer is not a bad tool for certain use cases. If you're managing Swarm clusters or need the full enterprise feature set, it's there. But for what I needed, it had become a liability. Slow, opaque about where your files actually live, and carrying way more weight than the job required.

Dockhand is the opposite. It does the things I actually use, it does them well, and it gets out of the way. The UI is clean without being oversimplified. The compose handling is transparent. The multi-environment support actually works the way you'd expect. And it's fast.

For anyone running a homelab with multiple Docker hosts and Compose stacks spread across environments, this is the tool. Stop tolerating whatever you're currently using if it's giving you grief. Dockhand is worth the 30-minute setup to find out.

---

*Dockhand GitHub: [github.com/Finsys/dockhand](https://github.com/Finsys/dockhand)*  
*Docs: [dockhand.pro/manual](https://dockhand.pro/manual)*
