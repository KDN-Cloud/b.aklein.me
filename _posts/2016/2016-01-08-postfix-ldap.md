---
title: 'Setup Postfix with LDAP'
date: 2016-01-08
---

# Intro
There are many ways to configure a virtual mail environment using postfix, but in this post I will describe the steps I took to configure postfix to work with OpenLDAP on a Linux host. The end goal was to utilize LDAP lookup tables for virtual domains, mailboxes and aliases.

### LMTP
First, instead of LDA I chose LMTP as my local delivery agent. As described in [Dovecot's LMTP wiki](http://wiki.dovecot.org/HowTo/PostfixDovecotLMTP), I changed `virtual_transport = dovecot` to = `lmtp:unix:private/dovecot-lmtp`

`master.cf` should also contain an external delivery method for LMTP similar to
```
dovecot   unix  -       n       n       -       -       pipe
  flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/lmtp -f ${sender} -d ${recipient}
```

Next, I included the following 8 virtual lines in `main.cf`

```
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000
virtual_minimum_uid = 5000
virtual_mailbox_domains = ldap:/etc/postfix/ldap/ldap-virtual-domains.cf
virtual_mailbox_maps = ldap:/etc/postfix/ldap/ldap-vmailbox.cf
virtual_alias_maps = ldap:/etc/postfix/ldap/ldap-aliases.cf
virtual_mailbox_limit = 512000000
virtual_mailbox_base = /home/vmail/
```
Notice I have specified `ldap:` instead of the more common `hash:` database with the absolute path to 3 files needed for domains, mailboxes and aliases.

### Domains
#### ldap-virtual-domains.cf
```
server_host = ldap://ldap.example.net/
search_base = ou=Domains,dc=example,dc=net
version = 3
bind = no
query_filter = (&(ObjectClass=dNSDomain)(dc=%s))
result_attribute = dc
```
Verify domain in LDAP queries successfully

`postmap -q domain1.net ldap:/etc/postfix/ldap/ldap-virtual-domains.cf`

will return `domain1.net`

If there is no result the domain may not be in LDAP. To add domains in LDAP see [this page](/multiple-mail-domains-in-ldap).

### Mailboxes
#### ldap-vmailbox.cf
```
server_host = ldap://ldap.example.net/
search_base = ou=Mail,dc=example,dc=net
version = 3
bind = no
query_filter = (&(objectclass=inetOrgPerson)(mail=%s))
result_attribute = mail
```
Verify user mailbox in LDAP queries successfully

`postmap -q johndoe@domain1.net ldap:/etc/postfix/ldap/ldap-vmailbox.cf`

will return `johndoe@domain1.net`

If there is no result make sure the email address exists as the _primary_ mail account and not a mail alias. See [this page](/official-email-accounts-aliases-in-ldap) on how you can add new mail user records in LDAP.

### Aliases
#### ldap-aliases.cf
```
server_host = ldap://ldap.example.net/
search_base = ou=Mail,dc=example,dc=net
version = 3
bind = no
query_filter = (&(objectclass=PostfixBookMailAccount)(mailAlias=%s))
result_attribute = mail
```
Verify user mail alias in LDAP queries successfully

`postmap -q superjohn@domain2.me ldap:/etc/postfix/ldap/ldap-aliases.cf`

will return `johndoe@domain1.net `

> Notice when you query an alias the primary email address is returned and not the actual alias.

If all queries return expected results for mail domains, mailboxes and aliases we can proceed with [configuring Dovecot to work with LDAP](/dovecot-with-ldap).
