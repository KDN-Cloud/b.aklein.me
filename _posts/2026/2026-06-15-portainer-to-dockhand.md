---
layout: post
toc: true
title: "Dock and Roll: Why I Ditched Portainer"
promote_deployonfriday: true
date: 2026-06-15
tags:
- docker
- docker-compose
- dockhand
- portainer
- portainer-alternative
- homelab
- self-hosted
- infrastructure
- container-management
- container-orchestration
- compose-stacks
- docker-management
- docker-ui
- docker-dashboard
- docker-stack
- docker-host
- docker-socket
- wireguard
- site-to-site
- site-to-site-vpn
- multi-environment
- multi-host
- remote-docker
- remote-management
- git-integration
- gitops
- webhook
- gitea
- github
- gitlab
- svelte
- sveltekit
- linux
- ubuntu
- debian
- proxmox
- proxmox-ve
- lxc
- vm
- authentik
- authentik-sso
- authentik-oidc
- authentik-integration
- authentik-oauth2
- authentik-provider
- oidc
- oidc-integration
- oauth2
- openid-connect
- sso
- sso-integration
- identity-provider
- idp
- mfa
- security
- hardening
- open-source
- bsl
- business-source-license
- privacy
- no-telemetry
- zero-telemetry
- ops
- devops
- sysadmin
- platform-engineering
- hawser
- docker-agent
- yaml
- env-file
- dot-env
- docker-security
- port-mapping
- sqlite
- lightweight
- migration
- lab
- network
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

Getting it running takes about 30 seconds. You can either spin it up with a quick `docker run` if you just want to kick the tires:

```bash
docker run -d \
  --name dockhand \
  --restart unless-stopped \
  -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v dockhand_data:/app/data \
  fnsys/dockhand:latest
```

Or the way I actually run it, with a proper compose file. The official example on [dockhand.pro](https://dockhand.pro/) is intentionally minimal to get you up fast, but for anything beyond a quick test you want more control than that. Here's my setup:

```yaml
services:
  dockhand:
    image: fnsys/dockhand:latest
    container_name: dockhand
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "8080:3000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data:/app/data
    environment:
      - ADMIN_USERNAME=${ADMIN_USERNAME}
      - ADMIN_PASSWORD=${ADMIN_PASSWORD}
      - TZ=America/Los_Angeles
```

> **Note:** Dockhand listens internally on port `3000`. The host-side port (`8080` above) is whatever you want to map it to on your system. I had other services already occupying `3000` on this host, so I mapped it to `8080` instead. If `3000` is free on your end, `"3000:3000"` works fine and matches the quick-start docs.

A few other things worth noting here compared to the bare minimum version. The socket is mounted read-only (`:ro`) which is a small but meaningful hardening step — Dockhand doesn't need write access to the socket itself. `no-new-privileges:true` prevents any process inside the container from escalating privileges via `setuid` or `setgid` binaries. Credentials come from a `.env` file rather than being hardcoded, and `TZ` is set so timestamps in the UI actually reflect your local time instead of UTC. No separate database container required by default, no multi-container setup to babysit on day one. Just up and running.

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

## Authentik SSO Integration

I run Authentik as my SSO/OIDC provider across the homelab, so getting Dockhand wired into it was a priority. The good news is Dockhand treats OIDC as a first-class feature and the setup is straightforward.

On the Authentik side, create a new OAuth2/OpenID Connect provider and application. The important part is the redirect URI — Dockhand expects:

```
https://your-dockhand-url/api/auth/oidc/callback
```

Make sure that's added to the allowed redirect URIs in your Authentik application config, otherwise the callback will get rejected.

On the Dockhand side, go to **Settings > Authentication > SSO / OIDC** and hit **Add provider**. You'll fill in four fields:

- **Name** — whatever you want to call it, I just used `authentik`
- **Issuer URL** — this comes from your Authentik application's OIDC discovery URL, formatted as `https://auth.yourdomain/application/o/<app-slug>/`
- **Client ID** and **Client Secret** — copy these from the Authentik provider you just created

Enable the provider, save, and then make sure the top-level **Authentication** toggle in Settings is set to **ON** — that's the master switch that enforces login for all access.

> **Note:** Authentication is off by default on a fresh Dockhand install. That's intentional — it lets you get in and configure things before locking yourself out. Just don't forget to flip it on before you expose the UI to anything beyond localhost.

Once it's set up, the Dockhand login page gives you the option to sign in with your OIDC provider or with a local user. I keep a local admin account as a fallback in case Authentik is ever unreachable, which has saved me at least once during an Authentik upgrade.

## Everything Else

Dockhand also has real-time log streaming, an interactive terminal for shell access into containers, a file browser for getting in and out of container filesystems, and real-time stats on CPU/memory/network/disk I/O.

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
