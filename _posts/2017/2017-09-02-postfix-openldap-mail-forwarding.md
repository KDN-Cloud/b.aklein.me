---
title: 'Mail forwarding with Postfix and OpenLDAP'
date: 2017-09-02
description: Configuring mail forwarding with Postfix and OpenLDAP
tags:
- Postfix
- LDAP
- Linux
- Email
---

For the most part mail forwarding is not too common within my Infrastructure. With Sieve deployed in my environment using the ManageSieve protocol - mail users are able to easily setup a redirect to their preferred email address. This all works fine, but I also wanted to have the ability to setup mail forwarding directly within OpenLDAP. 

Today I went ahead and pushed a [commit](https://github.com/variablenix/ldap-mail-schema/commit/e271d5f85b32fef887f341babc2f9028a7da07cb) for [postfix-book.schema](https://github.com/variablenix/ldap-mail-schema/blob/master/postfix-book.schema) to include a `mailForwardingAddress` attribute. The existing `PostfixBookMailForward` objectClass contains our `mailForwardingAddress` attribute, respectively.

## Forwarding
Assuming the schema is loaded into your environment, we can now tell Postfix to use LDAP mail forwarding. 

How?

We can create `ldap-forward.cf` in `/etc/postfix/ldap` with something like

```
server_host = ldap://ldap.example.com/
search_base = ou=Mail,dc=example,dc=com
version = 3
bind = no
query_filter = (&(|(mailAlias=%s)(mail=%s))(objectClass=PostfixBookMailForward))
result_attribute = mailForwardingAddress
```
The `query_filter` will match a user's primary mail address or any mail aliases while the `result_attribute` is the forwarded email address.

The `main.cf` file should have the `ldap-forward.cf` file defined in `virtual_alias_maps` using `proxy:ldap:/etc/postfix/ldap/ldap-forward.cf` 

```
virtual_alias_maps = ldap:/etc/postfix/ldap/ldap-aliases.cf,ldap:/etc/postfix/ldap/ldap-groups.cf proxy:ldap:/etc/postfix/ldap/ldap-forward.cf
```

To verify mail forwarding we can see that our forwarded email address _does_ get returned when querying the primary or alias email address.

```
postmap -q me@example ldap:/etc/postfix/ldap/ldap-forward.cf
forwarduser@somewhere
```