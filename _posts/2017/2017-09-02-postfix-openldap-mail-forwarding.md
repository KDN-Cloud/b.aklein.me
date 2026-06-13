---
title: 'Mail forwarding with Postfix and OpenLDAP'
date: 2017-09-02
lastmod: 2026-06-13
description: Configuring mail forwarding with Postfix and OpenLDAP
historical_warning_title: Historical mail routing warning
historical_warning_body: This post extends a classic Postfix plus OpenLDAP mail schema with LDAP-driven forwarding. The pattern still works, but it belongs to an older self-hosted mail model where directory schema changes were part of normal operations.
historical_warning_docs_url: https://www.postfix.org/LDAP_README.html
historical_warning_docs_label: official Postfix LDAP documentation
tags:
- postfix
- ldap
- linux
- openldap
- self-hosted-email
- mail-forwarding
- mail-server
- email-routing
- virtual-mailboxes
- virtual-aliases
- postfix-book-schema
- mail-schema
- homelab
- sysadmin
- mail-infrastructure
---

{% include historical-warning.html %}

Mail forwarding was never the primary way I handled redirection in this environment. With Sieve and ManageSieve already in place, users could create their own redirects without me touching the mail server. That covered most normal cases.

What I still wanted was directory-driven forwarding. In other words, the ability to say at the LDAP layer that a mailbox or alias should forward somewhere else, and have Postfix resolve that directly.

That meant extending the schema. I pushed a [commit](https://github.com/variablenix/ldap-mail-schema/commit/e271d5f85b32fef887f341babc2f9028a7da07cb) for [postfix-book.schema](https://github.com/variablenix/ldap-mail-schema/blob/master/postfix-book.schema) to include a `mailForwardingAddress` attribute. The `PostfixBookMailForward` objectClass is what carries that field.

## Forwarding

Once the schema is loaded, Postfix needs a lookup table for forwarding. I created `ldap-forward.cf` in `/etc/postfix/ldap` with this:

```
server_host = ldap://ldap.example.com/
search_base = ou=Mail,dc=example,dc=com
version = 3
bind = no
query_filter = (&(|(mailAlias=%s)(mail=%s))(objectClass=PostfixBookMailForward))
result_attribute = mailForwardingAddress
```
The `query_filter` will match a user's primary mail address or any mail aliases while the `result_attribute` is the forwarded email address.

That filter is doing exactly what it should:

- match the user's primary `mail` address
- also match any `mailAlias`
- only return records that implement `PostfixBookMailForward`

That way forwarding works whether the message was addressed to the main mailbox or to one of its aliases.

Then `main.cf` needs the new table in `virtual_alias_maps`, typically through the proxied LDAP lookup form:

```
virtual_alias_maps = ldap:/etc/postfix/ldap/ldap-aliases.cf,ldap:/etc/postfix/ldap/ldap-groups.cf proxy:ldap:/etc/postfix/ldap/ldap-forward.cf
```

I liked putting it after the normal aliases and groups lookups so the mail routing flow stayed predictable.

## Testing the lookup

To verify forwarding, query the table directly:

```
postmap -q me@example ldap:/etc/postfix/ldap/ldap-forward.cf
forwarduser@somewhere
```

If that address comes back, Postfix has what it needs.

If it returns nothing, check:

- whether the record actually has `mailForwardingAddress`
- whether the record includes the `PostfixBookMailForward` object class
- whether the base DN matches where the record really lives
- whether your alias or primary address is stored in the field the query expects

## Why I bothered with this

The real value here was policy at the directory layer. Instead of telling users to manage forwarding only through Sieve, I could also define forwarding behavior centrally from LDAP when needed.

That made sense for administrative accounts, utility mailboxes, and addresses that existed more as routing endpoints than as real inboxes.

## Where this fits now

The pattern still makes sense if you are already running the old Postfix + OpenLDAP schema. But if I were building from scratch today, I would be honest about the tradeoff: every new schema extension buys capability at the cost of more directory complexity.

This is one of those designs that feels elegant when you are deep in the mail stack and increasingly questionable when you step back and ask whether you still want to be the person maintaining all of it.
