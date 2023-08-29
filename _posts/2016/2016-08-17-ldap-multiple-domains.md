---
title: 'Adding multiple mail domains in LDAP'
published: true
date: 2016-08-17
tags:
- LDAP
- Email
- 'multiple domains'
---

# Intro
LDAP makes it a breeze to add multiple domain names you wish to serve email accounts with. Although I am describing how I configured multiple domains in my own environment using OpenLDAP - this should also work for other LDAP implementations. 

### Domains Organizational Unit
```
dn: ou=Domains,dc=domain1,dc=net
objectClass: organizationalUnit
objectClass: top
ou: Domains
description: Domains used for Postfix as its list of locally hosted domains
```
This LDIF will define our Domains Organizational Unit (OU). Add the LDIF with `ldapadd` so our domains have a container to live in.

### Adding Domains
```
dn: dc=domain1.net,ou=Domains,dc=domain1,dc=net
dc: domain1.net
objectClass: dNSDomain
objectClass: top

dn: dc=domain2.me,ou=Domains,dc=domain1,dc=net
dc: domain2.me
objectClass: dNSDomain
objectClass: top
```
After importing our domains from an LDIF we can now verify our 2 domains in LDAP get returned with the [postmap](http://www.postfix.org/postmap.1.html) command.

`$ postmap -q domain1.net ldap:/etc/postfix/ldap/ldap-virtual-domains.cf`

domain1.net

`$ postmap -q domain2.me ldap:/etc/postfix/ldap/ldap-virtual-domains.cf`

domain2.me