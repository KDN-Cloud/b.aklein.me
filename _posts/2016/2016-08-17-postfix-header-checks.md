---
title: 'Postfix Header Checks'
date: 2016-08-17
lastmod: 2026-06-12
description: Using header checks with Postfix to filter spam — updated for Postfix 3.11 and modern self-hosted mail stacks.
tags:
- postfix
- postfix-3-11
- mail-server
- email-security
- spam-filtering
- header-checks
- pcre
- self-hosted-email
- self-hosted
- linux
- sysadmin
- rspamd
- milter
- dkim
- spf
- dmarc
- mail-filtering
- dovecot
- homelab
- infrastructure
- smtp
- mta
---

> `[ STATUS: LOG UPDATED FOR 2026 RUNTIME ENVIRONMENT ]`

I originally wrote this in 2016 when I was actively tuning my own Postfix stack and leaning on PCRE header checks as a first line of defense against spam. The core mechanic is still completely valid in Postfix 3.11 — nothing about how `header_checks` works has fundamentally changed — but the context around it has shifted enough that this post needed a proper update.

The way it works: Postfix runs every incoming message header through a set of PCRE patterns you define in a separate file, evaluates them in order, and takes the first matching action. That action is usually `REJECT`, `WARN`, `DISCARD`, or `IGNORE`. You wire it up in `main.cf` like this:

```
header_checks = pcre:/etc/postfix/header_checks
```

Postfix 3.x also gives you a few variations that are worth knowing. `mime_header_checks` applies specifically to MIME headers. `nested_header_checks` handles headers inside attached messages, which matters more than people realize for catching forwarded spam. `milter_header_checks` applies PCRE checks to headers after a milter has already processed and potentially modified them, which is useful when you want a deterministic final pass on whatever Rspamd or another filter has added or rewritten. And `smtp_header_checks` does the same thing on the outbound path, which is handy for scrubbing internal headers before mail leaves your network.

One thing I noted in the original post that still holds: I had to comment out the `Precedence: bulk` reject rule because it was blocking legitimate auto-reply and out-of-office responses. Mailing lists and automated systems you actually want to receive use bulk precedence headers, so rejecting on that pattern alone generates false positives faster than it catches spam. If you want to filter bulk mail, do it by score in Rspamd rather than a blunt header match.

Here's a set of PCRE rules that hold up well in 2026:

```pcre
# Reject missing or malformed Message-ID — a reliable spam signal
/^Message-Id: <[^@]*>$/  REJECT Invalid Message-ID

# Reject From headers with no domain
/^From:.*@\s*$/  REJECT Malformed From address

# Catch obvious spam in subject lines
/^Subject:.*\b(V1agra|Cialis|enlargement)\b/i  REJECT Rejected due to spam content

# Reject forged Received headers claiming localhost as external origin
/^Received:.*from localhost.*by\s+((?!localhost)\S+)/  REJECT Forged Received header
```

One thing worth knowing: when you edit `/etc/postfix/header_checks`, you do not run `postmap` on it. `postmap` is for hash and btree lookup tables. PCRE files are read directly by Postfix. You just reload after making changes:

```bash
postfix reload
```

If you want to test a pattern before it touches live mail, `postmap` can do that in query mode:

```bash
postmap -q "Subject: Buy V1agra now" pcre:/etc/postfix/header_checks
```

It will tell you whether the pattern fires and what action Postfix would take. Good sanity check before pushing changes to production.

Where `header_checks` fits in a modern stack is worth being honest about. If you're running Rspamd as a milter alongside Postfix — which is the right call for any serious self-hosted setup — Rspamd is handling the actual intelligence: Bayesian filtering, DNSBL lookups, fuzzy hashing, SPF/DKIM/DMARC validation, scoring. The Postfix integration looks like this:

```
smtpd_milters = inet:localhost:11332
non_smtpd_milters = inet:localhost:11332
milter_default_action = accept
```

With that in place, raw PCRE header checks become a pre-filter for fast deterministic rejections rather than your primary defense. Things like obviously forged headers, patterns you want killed at the door before the message even gets scored, or organizational policies that need to be enforced at the MTA layer regardless of spam score. That's where they still earn their place.

The original Gist linked in this post is still on GitHub under the variablenix account if you want to see the 2016 ruleset for reference. I'd treat it as historical context. The patterns that were doing the real work back then are better handled by Rspamd now, and reaching for PCRE rules for things like RBL checks or sender reputation is the wrong tool for the job when you have a proper milter running.

If you're standing up a fresh Postfix instance, get Rspamd running first, then layer in `header_checks` only for the patterns that are deterministic enough that scoring adds no value. That split gives you speed where you need it and intelligence where you need that.
