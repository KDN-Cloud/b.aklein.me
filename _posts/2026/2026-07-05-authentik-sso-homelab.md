---
layout: post
#comments: false
toc: true
promote_deployonfriday: true
title: "Authentik SSO: Replacing Every Password Prompt in My Homelab"
date: 2026-07-04
author: Anthony Klein
description: "How I deployed Authentik as the SSO layer for my entire homelab, replacing per-service logins with a single identity provider across 20+ self-hosted services using proxy providers, OIDC, LDAP, and SCIM."
tags:
  - authentik
  - authentik-sso
  - authentik-proxy
  - authentik-oidc
  - authentik-oauth2
  - authentik-ldap
  - authentik-scim
  - authentik-provider
  - authentik-application
  - authentik-groups
  - authentik-flows
  - authentik-outpost
  - authentik-docker
  - authentik-homelab
  - authentik-self-hosted
  - authentik-nginx
  - authentik-npm
  - authentik-nginx-proxy-manager
  - authentik-forward-auth
  - authentik-mfa
  - authentik-totp
  - authentik-webauthn
  - authentik-passkey
  - authentik-user-management
  - authentik-admin
  - authentik-policy
  - authentik-binding
  - authentik-blueprint
  - authentik-embed-outpost
  - authentik-setup
  - authentik-install
  - authentik-configuration
  - authentik-integration
  - authentik-crowdsec
  - sso
  - single-sign-on
  - identity-provider
  - idp
  - oidc
  - oauth2
  - openid-connect
  - openid
  - ldap
  - scim
  - forward-auth
  - proxy-provider
  - homelab
  - homelab-sso
  - homelab-auth
  - homelab-oidc
  - homelab-identity
  - homelab-security
  - homelab-infrastructure
  - homelab-automation
  - homelab-docker
  - homelab-self-hosted
  - homelab-networking
  - homelab-linux
  - self-hosted
  - self-hosted-sso
  - self-hosted-auth
  - self-hosted-security
  - self-hosted-identity
  - selfhosted
  - docker
  - docker-compose
  - proxmox
  - proxmox-vm
  - nginx-proxy-manager
  - npm
  - reverse-proxy
  - cloudflare
  - cloudflare-tunnel
  - linux
  - linux-security
  - linux-sysadmin
  - ubuntu
  - debian
  - sysadmin
  - sre
  - infrastructure
  - devops
  - security
  - security-engineering
  - security-automation
  - iam
  - identity-access-management
  - access-control
  - mfa
  - two-factor-auth
  - totp
  - webauthn
  - passkey
  - password-management
  - grafana
  - prometheus
  - graylog
  - netbox
  - navidrome
  - jellyfin
  - portainer
  - uptime-kuma
  - crowdsec
  - wireguard
  - ansible
  - automation
  - kdn-lab
  - variablenix
  - networking
  - home-network
  - vlan
  - pfsense
  - homelab-observability
  - homelab-monitoring
  - fleet-security
  - infrastructure-as-code
  - gitops
---

Before Authentik, logging into my homelab looked like this: a different username and password for Grafana, another one for Graylog, another for NetBox, another for Navidrome, another for the Proxmox web UI, another for every other service I run. Some of them had their own user databases. Some of them had no auth at all and were just sitting behind an IP restriction hoping for the best.

That is fine when you have three services. It stops being fine when the number grows and you start losing track of which service uses which credentials, or when you add something new and realize you need to go create yet another account somewhere.

Authentik fixed all of that. One login, one place to manage users and access, one MFA flow, and every service behind it follows the same authentication pattern. Services that support OIDC get a real OAuth2 integration. Services that do not get a forward auth proxy provider that enforces authentication at the reverse proxy layer before a request even reaches the app.

This post covers how I set it up, how I think about the provider model, and what the integration looks like across different service types. It is not a step-by-step install tutorial. The [Authentik docs](https://docs.goauthentik.io/) handle that well. This is the operational picture from someone running it in production across a full homelab fleet.

{% include note.html content="This is part of an ongoing Authentik series on AK // SYS LOG. If you landed here looking for how to wire a specific app into Authentik, check the related posts at the bottom of this page." %}

---

## Why Authentik Over the Alternatives

The short answer: it runs well in Docker, the UI is usable, and it handles every integration pattern I actually need in one tool.

The longer answer is that the homelab SSO space has a few real options and none of them is objectively better. It depends on what you are trying to do.

Authelia is lightweight, fast, and resource-efficient. It is OpenID Certified as of 2025 with solid OAuth2/OIDC support. If your primary goal is forward auth with a small footprint and straightforward config, Authelia is a genuinely good choice and should not be dismissed. A lot of homelabs run it very happily.

Keycloak is enterprise-grade and capable of essentially anything in the identity space. The tradeoff is real configuration complexity and a heavier resource profile that can feel like overkill for a personal lab.

Authentik sits between the two. It is heavier than Authelia but lighter than Keycloak, and its protocol surface is broader: OAuth2/OIDC, SAML2, LDAP, RADIUS, and SCIM in one tool. Authelia covers OAuth2/OIDC and LDAP well but does not support SAML2, RADIUS, or SCIM provisioning. For a homelab that only needs forward auth and OIDC that difference probably does not matter. For a setup like mine where I wanted LDAP outpost support, SCIM, per-app flow control, and a visual admin UI for managing policies without editing config files, Authentik was the right fit.

If your needs are simpler, Authelia is worth serious consideration. If you want the broader protocol support and feature surface and do not mind the extra resource overhead, Authentik earns it.

The things that matter to me in a homelab context:

It runs entirely in Docker Compose with PostgreSQL and Redis as backing services. No external dependencies, no cloud callbacks, no phoning home. The whole stack lives on a dedicated Proxmox VM on the Lab VLAN.

The proxy provider model means I can put auth in front of anything, even apps with no native auth support at all. I run Nginx Proxy Manager so that is what the examples here use, but the same approach works with any reverse proxy that supports forward auth: Traefik, Caddy, HAProxy, or plain nginx.

OIDC support is first-class. Apps that do support OAuth2 get a real integration with groups claims, which means I can do role-based access without managing user lists inside each app separately.

LDAP outpost support lets me handle apps that only speak LDAP without running a separate directory service.

MFA works across all of it. One TOTP or WebAuthn enrollment protects every service behind Authentik, not just the ones that have their own MFA built in.

---

## The Stack

Authentik runs on a dedicated Proxmox VM on the Lab VLAN, reachable internally at `auth.lab.example.com`. It is not exposed directly to the internet. Public-facing access goes through Cloudflare and a Cloudflare tunnel, but the actual Authentik instance stays internal.

The Compose stack starts from the standard upstream layout. Here is the baseline to get up and running:

```yaml
services:
  postgresql:
    image: docker.io/library/postgres:16-alpine
    container_name: authentik-postgres
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    volumes:
      - ./postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${PG_PASS:?database password required}
      POSTGRES_USER: ${PG_USER:-authentik}
      POSTGRES_DB: ${PG_DB:-authentik}

  redis:
    image: docker.io/library/redis:alpine
    container_name: authentik-redis
    restart: unless-stopped
    command: --save 60 1 --loglevel warning
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 3s
    volumes:
      - ./redis:/data

  server:
    image: ghcr.io/goauthentik/server:2026.5.3
    container_name: authentik-server
    restart: unless-stopped
    command: server
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY}
      AUTHENTIK_ERROR_REPORTING__ENABLED: false
    volumes:
      - ./media:/media
      - ./custom-templates:/templates
    ports:
      - "9000:9000"
      - "9443:9443"
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy

  worker:
    image: ghcr.io/goauthentik/server:2026.5.3
    container_name: authentik-worker
    restart: unless-stopped
    command: worker
    user: root
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY}
      AUTHENTIK_ERROR_REPORTING__ENABLED: false
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./media:/media
      - ./certs:/certs
      - ./custom-templates:/templates
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
```

Keep all secrets in a `.env` file that does not go anywhere near your git repo. `AUTHENTIK_SECRET_KEY` should be a long random string generated once and never changed unless you want to invalidate every active session.

### How I Beefed Mine Up

The vanilla compose gets you running. Here is how my production version differs, with the things I think are worth carrying into any real deployment:

```yaml
services:
  postgresql:
    env_file:
      - .env
    environment:
      POSTGRES_DB: ${PG_DB:-authentik}
      POSTGRES_PASSWORD: ${PG_PASS:?database password required}
      POSTGRES_USER: ${PG_USER:-authentik}
    healthcheck:
      interval: 30s
      retries: 5
      start_period: 20s
      test:
        - CMD-SHELL
        - pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}
      timeout: 5s
    image: docker.io/library/postgres:16-alpine
    restart: unless-stopped
    volumes:
      - database:/var/lib/postgresql/data
    deploy:
      resources:
        limits:
          memory: 512M

  server:
    command: server
    labels:
      - "crowdsec.enable=true"
      - "crowdsec.labels.type=authentik"
    depends_on:
      postgresql:
        condition: service_healthy
    env_file:
      - .env
    environment:
      AUTHENTIK_LOG_LEVEL: info
      AUTHENTIK_LOG_JSON: "false"
      AUTHENTIK_LISTEN__TRUSTED_PROXY_CIDRS: 192.168.0.0/16,10.0.0.0/8,172.16.0.0/12
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY:?secret key required}
      AUTHENTIK_CSRF_TRUSTED_ORIGINS: "https://auth.lab.example.com"
      AUTHENTIK_HOST: "https://auth.lab.example.com"
      AUTHENTIK_LISTEN__METRICS: 0.0.0.0:9300
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:${AUTHENTIK_TAG:-2026.5.3}
    ports:
      - ${COMPOSE_PORT_HTTP:-9009}:9000
      - ${COMPOSE_PORT_HTTPS:-9443}:9443
      - "9300:9300"
    restart: unless-stopped
    shm_size: 512mb
    volumes:
      - ./data/media:/media
      - ./data/custom-templates:/templates
      - ./data:/data
    deploy:
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 512M

  worker:
    command: worker
    labels:
      - "crowdsec.enable=true"
      - "crowdsec.labels.type=authentik"
    depends_on:
      postgresql:
        condition: service_healthy
    env_file:
      - .env
    environment:
      AUTHENTIK_LOG_LEVEL: info
      AUTHENTIK_LOG_JSON: "false"
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY:?secret key required}
      AUTHENTIK_CSRF_TRUSTED_ORIGINS: "https://auth.lab.example.com"
      AUTHENTIK_HOST: "https://auth.lab.example.com"
      AUTHENTIK_LISTEN__METRICS: 0.0.0.0:9300
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:${AUTHENTIK_TAG:-2026.5.3}
    ports:
      - "9301:9300"
    restart: unless-stopped
    shm_size: 512mb
    user: root
    volumes:
      - ./data/media:/media
      - ./data/certs:/certs
      - ./data/custom-templates:/templates
      - ./data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      resources:
        limits:
          memory: 1.5G
        reservations:
          memory: 512M

volumes:
  database:
    driver: local
```

A few things worth calling out here. The image reference uses `${AUTHENTIK_TAG}` instead of a hardcoded version, which makes upgrades a one-line `.env` change rather than a compose file edit. `AUTHENTIK_LISTEN__TRUSTED_PROXY_CIDRS` tells Authentik which upstream IPs to trust for real client IP forwarding. Without this, everything behind a reverse proxy looks like it comes from the proxy itself. `AUTHENTIK_LISTEN__METRICS` exposes a Prometheus metrics endpoint on port `9300`, which I scrape via Grafana. The CrowdSec labels wire both containers into the fleet security mesh so the local agent picks up Authentik-sourced events. `shm_size` gives the containers a larger shared memory allocation which helps under load. The `deploy.resources` limits cap memory so a runaway process does not take down the whole VM. If you run a SIEM like Graylog, the `logging` block with a GELF driver is where you would add centralized log forwarding for both the server and worker containers.

---

## How I Think About Providers

This is the part most guides skip. They show you how to create a provider for one specific app. What they do not explain is the mental model for deciding which provider type to use.

Authentik has a few provider types. The two I use most are proxy providers and OAuth2/OIDC providers.

**Proxy providers** are for apps that have no native auth support or where you do not want to bother with a full OIDC integration. Authentik sits in front of the app at the reverse proxy layer. When a request comes in, NPM checks with Authentik's outpost. If the user is not authenticated, they get redirected to the Authentik login flow. Once authenticated, the request passes through to the app. The app itself never sees any of this. From its perspective, requests just arrive.

This is the right choice for internal tools, admin UIs, and anything where you want auth without touching the app's own config.

**OAuth2/OIDC providers** are for apps that speak OIDC natively. The app handles its own authentication flow using Authentik as the identity provider. You get real group-based role mapping, proper token handling, and the app can show user-specific content based on claims. This requires configuring both Authentik and the app itself.

The rule I use: if the app has OIDC support in its docs, use an OIDC provider. If it does not, use a proxy provider.

---

## The Embedded Outpost

The proxy provider model requires an outpost, which is the component that handles the actual forward auth requests from your reverse proxy. Authentik ships with an embedded outpost that handles this without needing to run a separate container.

For a homelab setup where everything is on the same network, the embedded outpost is usually enough. The one thing to know is that the outpost needs to be reachable from NPM. In my setup, NPM is on the same Proxmox VM as Authentik and they talk over localhost. The forward auth URL in NPM points to the Authentik server's internal port:

```
http://localhost:9000/outpost.goauthentik.io/auth/nginx
```

If your NPM is on a different host, use the Authentik host IP instead of localhost.

---

## Nginx Proxy Manager Forward Auth Setup

For every proxy provider app, the NPM config follows the same pattern. In the proxy host settings for the service under **Advanced**:

```nginx
location /outpost.goauthentik.io {
    proxy_pass          http://localhost:9000/outpost.goauthentik.io;
    proxy_set_header    Host $host;
    proxy_set_header    X-Original-URL $scheme://$http_host$request_uri;
    add_header          Set-Cookie $auth_cookie;
    auth_request_body   $body;
}

location / {
    auth_request        /outpost.goauthentik.io/auth/nginx;
    error_page          401 = @goauthentik_proxy_signin;
    auth_request_set    $auth_cookie $upstream_http_set_cookie;
    add_header          Set-Cookie $auth_cookie;
    auth_request_set    $authentik_username $upstream_http_x_authentik_username;
    auth_request_set    $authentik_groups $upstream_http_x_authentik_groups;
    auth_request_set    $authentik_email $upstream_http_x_authentik_email;
    auth_request_set    $authentik_name $upstream_http_x_authentik_name;
    auth_request_set    $authentik_uid $upstream_http_x_authentik_uid;
    proxy_set_header    X-authentik-username $authentik_username;
    proxy_set_header    X-authentik-groups $authentik_groups;
    proxy_set_header    X-authentik-email $authentik_email;
    proxy_set_header    X-authentik-name $authentik_name;
    proxy_set_header    X-authentik-uid $authentik_uid;
}

location @goauthentik_proxy_signin {
    internal;
    add_header          Set-Cookie $auth_cookie;
    return 302          /outpost.goauthentik.io/start?rd=$request_uri;
}
```

That block is the same for every proxy provider service. Copy, paste, done.

The Authentik proxy provider for each service needs the external URL set to match what NPM is serving. Create the application in Authentik first, then create the proxy provider pointing at the internal service URL, then apply the NPM config above.

---

## OIDC Integrations

Services that support OIDC get a native integration. The setup is always the same on the Authentik side: create an OAuth2/OpenID provider, create an application, copy the client ID, client secret, and issuer URL. What varies is how each app consumes those values.

For every OIDC provider you create, the issuer URL follows the same pattern:

```
https://auth.lab.example.com/application/o/<app-slug>/
```

The trailing slash is required. The app fetches `/.well-known/openid-configuration` from that URL to discover endpoints automatically. You can verify it directly in a browser before touching any app config. If it returns a JSON blob with `authorization_endpoint`, `token_endpoint`, and `jwks_uri`, the provider is set up correctly.

A few integrations from my fleet worth calling out.

**Grafana** has solid OIDC support. In `grafana.ini` or the equivalent env vars:

```ini
[auth.generic_oauth]
enabled = true
name = Authentik
client_id = <client-id>
client_secret = <client-secret>
scopes = openid profile email groups
auth_url = https://auth.lab.example.com/application/o/grafana/authorize/
token_url = https://auth.lab.example.com/application/o/token/
api_url = https://auth.lab.example.com/application/o/userinfo/
role_attribute_path = contains(groups[*], 'grafana-admins') && 'Admin' || 'Viewer'
```

The `role_attribute_path` maps Authentik group membership directly to Grafana roles. Users in `grafana-admins` get Admin. Everyone else gets Viewer. No user management needed inside Grafana.

**Graylog** supports OAuth2 through its enterprise auth plugin, but the community version is more limited. I run it behind a proxy provider rather than a native OIDC integration. It gets Authentik's forward auth in front of it and Graylog's own internal auth handles sessions after that. Not ideal but functional.

**NetBox** has Django OAuth2 support. The config goes in `configuration.py`:

```python
SOCIAL_AUTH_OIDC_ENDPOINT = 'https://auth.lab.example.com/application/o/netbox/'
SOCIAL_AUTH_OIDC_KEY = '<client-id>'
SOCIAL_AUTH_OIDC_SECRET = '<client-secret>'
```

Users land on the Authentik flow and come back to NetBox authenticated. Group-to-permission mapping works through NetBox's own permission model once the user account exists.

**CrowdSec Web UI** gets a native OIDC integration as of the version that shipped authentication. I covered that in detail in the [CrowdSec Web UI OIDC post](https://b.aklein.me/2026/06/29/crowdsec-web-ui-oidc-authentik.html) if you want the specifics.

---

## LDAP Outpost

Some apps only speak LDAP and have no OIDC or OAuth2 support. Rather than running a separate directory service like OpenLDAP just for those apps, Authentik's built-in LDAP outpost exposes a read-only LDAP interface backed by Authentik's own user and group data.

The LDAP outpost is not enabled by default. You need to create it explicitly.

Go to **Admin > Applications > Outposts > Create**. Set the type to **LDAP** and assign it to the applications you want to serve over LDAP. Authentik will spin up an LDAP listener on port 389 (or 636 for LDAPS) inside your Docker network.

Add the LDAP outpost container to your Compose stack:

```yaml
  ldap:
    image: ghcr.io/goauthentik/ldap:2026.5.3
    container_name: authentik-ldap
    restart: unless-stopped
    environment:
      AUTHENTIK_HOST: http://server:9000
      AUTHENTIK_INSECURE: "true"
      AUTHENTIK_TOKEN: <outpost-token>
    ports:
      - "389:3389"
      - "636:6636"
```

The `AUTHENTIK_TOKEN` comes from the outpost you created in the UI. Go to the outpost detail page and copy the service account token.

Once running, apps that need LDAP point at the Authentik LDAP outpost host instead of a dedicated directory server. The base DN follows the pattern `dc=ldap,dc=goauthentik,dc=io` by default, though you can configure a custom base DN in the outpost settings.

Users authenticate with their Authentik credentials. Groups map to LDAP groups. For most homelab apps that still require LDAP (some older monitoring tools, certain mail clients, legacy internal apps), this covers it without standing up a whole separate directory.

---

## MFA Across the Fleet

This is one of the things I appreciate most about running a central identity provider. Before Authentik, MFA was service-by-service. Some services had it, some did not, some had it but only for certain account types.

Now MFA enforcement lives in Authentik's authentication flow. Every service behind Authentik inherits the same MFA requirement. One TOTP enrollment or WebAuthn passkey registration covers everything.

MFA is not enabled by default in Authentik. You have to wire it into the authentication flow yourself.

### Enabling TOTP

Under **Admin > Directory > Authenticator TOTP Devices**, you will not find much until you add the stage. The actual setup happens in **Flows and Stages**.

Go to **Flows and Stages > Stages > Create** and add an **Authenticator TOTP Stage**. Give it a name like `totp-setup-stage`. This is the stage that walks users through scanning a QR code with their authenticator app on first login.

Then go to **Flows and Stages > Stages > Create** again and add an **Authenticator Validation Stage**. This is the stage that prompts for the TOTP code on subsequent logins. In its configuration, set **Device Classes** to include TOTP (and WebAuthn if you want both).

Now bind both stages to your authentication flow. Under **Flows and Stages > Flows**, open `default-authentication-flow` and go to **Stage Bindings**. The order matters:

1. `default-authentication-identification` (identification)
2. `default-authentication-password` (password)
3. `totp-setup-stage` (enrollment, only shown if the user has no device yet)
4. `default-authentication-mfa-validation` (validation, prompts for code)

Set the TOTP setup stage binding to **Not configured action: Force the user to configure** so first-time users are walked through enrollment automatically.

### Enabling WebAuthn / Passkeys

The process is the same pattern. Create an **Authenticator WebAuthn Stage** for enrollment and add it as a stage binding in the flow. WebAuthn works for hardware security keys and passkeys stored in a password manager or device authenticator.

In the Authenticator Validation Stage config, add WebAuthn to the **Device Classes** list alongside TOTP. Users can then enroll either method or both.

### Separate Flows for Internal vs External

For homelab use I run two authentication flows: one with MFA required for anything externally reachable, and a simpler one without the MFA validation stage for internal-only services. Applications get bound to the appropriate flow in their Authentik application settings under **Policy / Bindings > Flows**.

The split means I am not prompted for a TOTP code every time I open an internal tool from the LAN, but anything going through Cloudflare requires full MFA. That balance has worked well day to day.

---

## Groups and Access Control

Every user in Authentik belongs to one or more groups. Those groups are what you use to control who can access what.

The model I run:

- `homelab-admins` gets full access to everything
- `homelab-users` gets access to media and personal services
- `homelab-readonly` gets viewer access to monitoring services

Groups are assigned in Authentik's user management UI. Applications get policy bindings that reference groups. If a user is not in a group that has access to an application, Authentik denies the auth request before it even reaches the app.

A group binding on an application looks like this in the UI: go to the application, open **Policy / Bindings**, click **Bind existing policy**, select **Group**, choose your group, and set the order. You can stack multiple bindings with OR logic (any matching group grants access) or AND logic (all bindings must pass). For homelab use, OR is almost always what you want.

For OIDC integrations, groups are passed as a claim in the ID token. Apps that support group-based roles (Grafana, NetBox) can map those claims directly without any additional config inside the app's user management. For proxy provider apps, Authentik passes group info in request headers that the app can use or ignore.

---

## Local Fallbacks and Break-Glass Accounts

Most things in my fleet authenticate through Authentik, but every critical piece of infrastructure has a local fallback account that bypasses it entirely. That is intentional.

Proxmox uses Authentik via OpenID Connect as the primary login path. There is a dedicated `authentik` realm configured under Datacenter > Permissions > Realms, and users like `akadmin@authentik` and `staff@domain.me@authentik` log in through the Authentik OIDC flow directly from the Proxmox login screen. The local `root@pam` account stays active as a break-glass fallback. If Authentik goes down I still need to be able to reach the hypervisor to diagnose it. Locking yourself out of Proxmox because your SSO is unavailable is not a position you want to be in.

pfSense uses the Authentik LDAP outpost as the primary authentication path, with a local admin account kept active for the same reason.

The Authentik admin UI itself is outside Authentik's own auth flow. There is a local `akadmin` account that bypasses everything. That account exists for one reason: getting back in when something goes wrong with the flows or the database.

Services that are completely internal and not reachable from outside the lab VLAN without being on the VPN first get less scrutiny. The VPN itself is the access control layer there.

Everything public-facing goes through Authentik with MFA required, no exceptions.

---

## Operational Notes

A few things that come up once you are running this day to day.

**Session length.** The default token expiry in Authentik is relatively short. For homelab use where you want to stay logged in across browser sessions, bump the token expiry on your applications. Under the provider settings, the `Access Token Validity` and `Refresh Token Validity` fields control this.

**Backup the PostgreSQL database.** Everything lives in Postgres: users, groups, flows, provider configs, OIDC secrets. A pg_dump on a schedule is non-negotiable. Losing the database means recreating every integration from scratch.

A simple cron backup:

```bash
#!/usr/bin/env bash
BACKUP_DIR="/home/user/backups/authentik"
DATE=$(date +%Y-%m-%d)
mkdir -p "$BACKUP_DIR"
docker exec authentik-postgres pg_dump -U authentik authentik | gzip > "$BACKUP_DIR/authentik-$DATE.sql.gz"
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +14 -delete
```

Run it nightly and keep two weeks of retention. The dump is small. A full Authentik database with dozens of integrations, users, and flows is typically under 5MB compressed.

The pg_dump script is the portable, stack-agnostic approach and worth having regardless of your broader backup strategy. In my case Proxmox handles daily snapshots of all VMs and LXC containers to a UniFi UNAS Pro over SMB/CIFS, which covers the Authentik VM as a whole. The database dump gives me a lightweight, restore-anywhere option that does not require spinning up the full VM to recover a single integration. Both layers serve different recovery scenarios.

If you are storing Proxmox backups on a NAS over SMB/CIFS and it is working reliably, there is no compelling reason to switch to NFS for this use case. NFS has slightly lower overhead and is the more traditional Linux-to-NAS choice, but the practical difference for daily backup jobs is negligible. NFS makes more sense if you are doing live storage migration or direct VM disk access from the NAS rather than just backup dumps.

**Watch the worker container.** The worker handles background tasks including certificate rotation, policy evaluation, and some flow processing. If the worker container goes unhealthy, things start breaking in non-obvious ways. I have it in my Uptime Kuma internal monitoring alongside the server container.

**The embedded outpost needs to be configured for your external URL.** Go to **Admin > Outposts** and check the embedded outpost's configuration. The `authentik_host` setting needs to point to the externally reachable URL of your Authentik instance, not `localhost`. This is the URL that redirect flows use, and if it is wrong, post-auth redirects go nowhere.

---

## The Before and After

Before: 20+ services with independent credential stores, inconsistent MFA coverage, no central way to revoke access when needed, and a mental overhead of remembering which password goes where.

After: one login, one MFA enrollment, consistent access control through group membership, and if I need to remove someone's access to everything I manage one user record.

For a solo homelab that is probably overkill in the strictest sense. But the operational simplicity of having one auth layer across everything is genuinely worth the initial setup cost. The first time I needed to rotate credentials on several services at once and realized it was one change in one place, the investment paid for itself.

Authentik is one of the services in my fleet that I genuinely would not want to run without anymore.

---

This is part of an ongoing series covering how I run Authentik across the KDN Lab fleet. Related posts:

- [CrowdSec Web UI Now Has Native OIDC. Here's How to Wire It Into Authentik.](https://b.aklein.me/2026/06/29/crowdsec-web-ui-oidc-authentik.html)
- [Dock and Roll: Why I Ditched Portainer](https://b.aklein.me/2026/06/15/portainer-to-dockhand.html) (includes Dockhand OIDC integration with Authentik)

