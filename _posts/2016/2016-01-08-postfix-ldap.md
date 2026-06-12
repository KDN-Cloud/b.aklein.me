---
title: 'Postfix with LDAP'
promote_deployonfriday: true
description: How I configured Postfix to use LDAP lookup tables for virtual domains, mailboxes, and aliases, plus what still makes sense in a modern mail stack.
date: 2016-01-08
lastmod: 2026-06-12
historical_warning_title: Historical config warning
historical_warning_body: The overall Postfix plus LDAP lookup pattern here still makes sense, but the surrounding mail stack and security defaults have changed a lot since 2016. Treat the examples below as working reference for a classic mail schema, not as a blind copy-paste for a new deployment.
historical_warning_docs_url: https://www.postfix.org/LDAP_README.html
historical_warning_docs_label: official Postfix LDAP documentation
tags:
- postfix
- ldap
- openldap
- lmtp
- dovecot
- virtual-mailboxes
- virtual-domains
- mail-aliases
- self-hosted-email
- mail-server
- smtp
- linux
- sysadmin
- homelab
- directory-services
- mail-routing
- mail-infrastructure
- vmail
- postfix-book-schema
- maildir
---

> `[ STATUS: LOG UPDATED FOR 2026 RUNTIME ENVIRONMENT ]`

{% include historical-warning.html %}

There are a lot of ways to build a virtual mail stack with Postfix. This is the path I took when I wanted Postfix to stop depending on local flat files and instead use LDAP lookup tables for virtual domains, mailboxes, and aliases. At the time that gave me one directory backend for the whole mail environment, which was exactly what I wanted.

That design still holds up conceptually in 2026. Postfix is still very good at using LDAP as a lookup source. What changed is everything around it: safer transport defaults, better identity platforms, and a much stronger argument for keeping the lookup model simple. If you're building fresh today, the part to think hardest about isn't Postfix itself. It's whether you really want a classic LDAP-backed mail schema, or whether you want a more modern identity layer and a dedicated mail platform.

## The goal

The end state here was straightforward:

- Postfix should recognize which domains I host.
- Postfix should confirm whether a mailbox exists.
- Postfix should resolve aliases back to the real mailbox.
- Dovecot should handle final delivery over LMTP instead of using a legacy LDA path.

That split still feels right to me. Let Postfix decide where the message goes. Let Dovecot handle mailbox delivery.

## LMTP instead of LDA

I chose LMTP as the local delivery path rather than an older LDA-style setup. That is still the better direction. Postfix hands the message off cleanly and Dovecot stays responsible for mailbox semantics, quota logic, sieve, and delivery behavior.

In `main.cf` I changed the transport to Dovecot LMTP:

```conf
virtual_transport = lmtp:unix:private/dovecot-lmtp
```

On older deployments you will also see `master.cf` service definitions that shell out to Dovecot's LMTP binary. That worked, but for a modern setup I would keep the intent simple: use Dovecot's LMTP service over the unix socket and avoid cleverness unless you actually need it.

## Core virtual mailbox settings

These were the main virtual mailbox lines in `main.cf`:

```conf
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000
virtual_minimum_uid = 5000
virtual_mailbox_domains = ldap:/etc/postfix/ldap/ldap-virtual-domains.cf
virtual_mailbox_maps = ldap:/etc/postfix/ldap/ldap-vmailbox.cf
virtual_alias_maps = ldap:/etc/postfix/ldap/ldap-aliases.cf
virtual_mailbox_limit = 512000000
virtual_mailbox_base = /home/vmail/
```

The important part is really the three LDAP tables:

- `virtual_mailbox_domains`
- `virtual_mailbox_maps`
- `virtual_alias_maps`

Those are what let Postfix ask LDAP whether a domain exists, whether a mailbox exists, and whether an alias should be rewritten to some other destination.

If you're new to this pattern, build and test each lookup independently before you trust the full mail flow. That advice from 2016 still matters.

## Virtual domains

The domain lookup file looked like this:

### `ldap-virtual-domains.cf`

```conf
server_host = ldap://ldap.example.net/
search_base = ou=Domains,dc=example,dc=net
version = 3
bind = no
query_filter = (&(objectClass=dNSDomain)(dc=%s))
result_attribute = dc
```

Then I verified a hosted domain directly with `postmap`:

```bash
postmap -q domain1.net ldap:/etc/postfix/ldap/ldap-virtual-domains.cf
```

That should return:

```text
domain1.net
```

If it returns nothing, Postfix doesn't know that domain exists. In my case that usually meant the LDAP record was missing or the search base was wrong.

## Virtual mailboxes

For mailbox lookups:

### `ldap-vmailbox.cf`

```conf
server_host = ldap://ldap.example.net/
search_base = ou=Mail,dc=example,dc=net
version = 3
bind = no
query_filter = (&(objectClass=inetOrgPerson)(mail=%s))
result_attribute = mail
```

Verification was the same idea:

```bash
postmap -q johndoe@domain1.net ldap:/etc/postfix/ldap/ldap-vmailbox.cf
```

Expected result:

```text
johndoe@domain1.net
```

If that query fails, the address either does not exist, exists only as an alias, or your LDAP filter is not matching the object class you think it is.

## Virtual aliases

Alias lookups were handled separately:

### `ldap-aliases.cf`

```conf
server_host = ldap://ldap.example.net/
search_base = ou=Mail,dc=example,dc=net
version = 3
bind = no
query_filter = (&(objectClass=PostfixBookMailAccount)(mailAlias=%s))
result_attribute = mail
```

Then:

```bash
postmap -q superjohn@domain2.me ldap:/etc/postfix/ldap/ldap-aliases.cf
```

Expected result:

```text
johndoe@domain1.net
```

That behavior matters. When you query the alias, Postfix needs the real mailbox destination back, not the alias itself.

## What I would change if I were building this today

The lookup pattern is still fine. The defaults are what I would revisit.

First, I would not leave LDAP transport loose unless there was a very good reason. If this is crossing a network, use LDAPS or STARTTLS and treat the directory like infrastructure that deserves the same care as the mail server itself.

Second, I would prefer a dedicated read-only bind account over anonymous lookups. Anonymous binds were easy for homelab testing, but a real deployment should be deliberate about ACLs and exactly what Postfix is allowed to read.

Third, I would keep testing table lookups with `postmap -q` until they are boring. Postfix's own docs still push this habit, and for good reason. If your LDAP query result doesn't exactly match what a local file lookup would have returned, the mail flow gets weird fast.

## Where modern identity platforms fit and where they do not

This is where a lot of people get tripped up now.

Dovecot can authenticate against LDAP-backed identity providers pretty comfortably, including an Authentik LDAP outpost, because that problem is fundamentally "find user, validate auth, return mailbox fields."

Postfix virtual lookup tables are a different kind of problem. Postfix is not asking "who is this user?" It is asking:

- is this domain hosted here?
- does this mailbox exist?
- what is the destination for this alias?

That means Postfix still wants a mail-aware directory model. If you already have a classic OpenLDAP schema for mail, great. If you don't, forcing a general identity provider to behave like a mail routing database can turn into more trouble than it is worth.

That is one of the big reasons I eventually cooled on running my own mail stack long term. The stack still works. The overhead just keeps accumulating.

## Practical 2026 take

If you are maintaining an existing Postfix + LDAP environment, this model is still valid. Tighten the transport security, use a least-privilege bind account, test every table with `postmap`, and keep Dovecot on LMTP for final delivery.

If you are building fresh, I would ask a more basic question first: do you actually want to own the directory schema, alias model, mailbox routing, auth layer, delivery path, and spam stack yourself in 2026? If the answer is yes, this pattern still works. If the answer is "probably not," listen to that.

If all three LDAP queries return the expected results for domains, mailboxes, and aliases, the next step is Dovecot auth and mailbox handling. That piece is here: [Dovecot with LDAP](/2016/08/08/dovecot-ldap.html)
