---
title: 'Adding multiple mail domains in LDAP'
description: Setup multiple domains in OpenLDAP.
date: 2016-08-17
lastmod: 2026-06-13
historical_warning_title: Historical mail schema warning
historical_warning_body: The multi-domain lookup model here still works, but this post assumes a classic OpenLDAP-backed Postfix mail schema from 2016. Use it as historical reference and re-check the surrounding lookup, transport, and identity assumptions before putting it into production.
historical_warning_docs_url: https://www.postfix.org/LDAP_README.html
historical_warning_docs_label: official Postfix LDAP documentation
tags:
- ldap
- openldap
- multiple-domains
- mail-server
- virtual-mail-domains
- domain-management
- postfix
- self-hosted-email
- virtual-domains
- virtual-mailboxes
- mail-routing
- mail-infrastructure
- dnsdomain
- homelab
- linux
- sysadmin
---

{% include historical-warning.html %}

This was the point where the mail stack stopped being “one domain on one box” and started acting more like an actual hosted mail environment. Postfix needs a way to answer a very simple question before it does anything useful: is this domain one of mine or not?

In my setup, LDAP answered that question.

## Why I separated domains into their own OU

I kept mail domains in a dedicated `Domains` organizational unit. That made the lookup logic simple and kept the records clean. Postfix did not need to search through user entries or alias records just to decide whether `example.net` belonged to the local system.

The OU looked like this:

```ldif
dn: ou=Domains,dc=domain1,dc=net
objectClass: organizationalUnit
objectClass: top
ou: Domains
description: Domains used for Postfix as its list of locally hosted domains
```

Add that with `ldapadd` and you have a container for every hosted domain the mail stack should recognize.

## Adding hosted domains

Each hosted domain was its own `dNSDomain` record:

```ldif
dn: dc=domain1.net,ou=Domains,dc=domain1,dc=net
dc: domain1.net
objectClass: dNSDomain
objectClass: top

dn: dc=domain2.me,ou=Domains,dc=domain1,dc=net
dc: domain2.me
objectClass: dNSDomain
objectClass: top
```

That is all Postfix really needed on the directory side. If the domain exists in LDAP, the lookup returns it. If it does not, Postfix treats it as unknown.

## Where Postfix uses this

This record set backs the `virtual_mailbox_domains` lookup in Postfix, usually through something like:

```conf
virtual_mailbox_domains = ldap:/etc/postfix/ldap/ldap-virtual-domains.cf
```

And the LDAP lookup file typically points at the `Domains` OU with a filter that matches the requested domain name.

That separation was one of the cleaner parts of the whole OpenLDAP mail design. Domains in one place, users in another, aliases somewhere else, and Postfix asking each table for exactly one thing.

## Testing with `postmap`

Before trusting live mail flow, verify the domain lookups directly:

```bash
postmap -q domain1.net ldap:/etc/postfix/ldap/ldap-virtual-domains.cf
```

Expected result:

```text
domain1.net
```

And:

```bash
postmap -q domain2.me ldap:/etc/postfix/ldap/ldap-virtual-domains.cf
```

Expected result:

```text
domain2.me
```

If you get no output, Postfix is not seeing the domain. In practice that usually means one of three things:

- the LDAP record does not exist
- the search base is wrong
- the query filter does not match the object class or attribute you stored

This is exactly why `postmap -q` matters. You can prove the lookup works before ever sending a real message.

## Why this mattered operationally

Once the domain list lived in LDAP, adding a new hosted domain stopped being a Postfix file-edit exercise. You add the domain record, verify it with `postmap`, and the lookup layer already knows what to do.

That made multi-domain hosting feel much less brittle. It also made the structure easier to reason about when debugging:

- domain lookup confirms whether Postfix owns the domain
- mailbox lookup confirms whether the user exists
- alias lookup confirms whether the address rewrites elsewhere

Those three checks tell you very quickly where a delivery problem actually lives.

## Where this fits now

The basic model still makes sense. Even in 2026, separating domain lookups from mailbox and alias lookups is clean design. What I would question now is not the modeling, but the amount of directory infrastructure you are willing to maintain just to support it.

If you already run the classic OpenLDAP mail schema, this pattern is still fine. If you are building from scratch, think hard before signing yourself up for the full operational overhead just to host a handful of domains.
