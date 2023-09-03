---
title: 'Adding official email accounts and aliases in LDAP'
description: Setup an official email account and email aliases with LDAP
date: 2016-8-15
tags:
- LDAP
- Mail
- Accounts
- Aliases
---

# Intro
This post will touch on what objectClass and attributes I used specifically for OpenLDAP mail user records. I like the idea of keeping things well organized and with this simple structure I'm keeping the `People` and `Mail` containers separate. As a result, user records in the Mail organizational unit will have mail specific attributes not found in `People` user records. 

For the attributes to work I needed to have [postfix-book.schema](https://github.com/variablenix/ldap-mail-schema/blob/master/postfix-book.schema) loaded into LDAP.

### Import Mail Account
```
dn: uid=jdoe,ou=Mail,dc=domain1,dc=net
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: PostfixBookMailAccount
uid: jdoe
cn: John Doe
sn: Doe
mailEnabled: TRUE
mailAlias: alias1@domain1
mailAlias: alias2@domain2
mailAlias: alias3@domain1
mailAlias: alias4@domain2
mailUidNumber: 5000
mailGidNumber: 5000
mail: johndoe@domain1
description: John Doe's mail account
userPassword: {SSHA}lFXu8SajJaj+vEk99SvsBa+sRLmLfiRV
mailHomeDirectory: /home/vmail/domain1.net/johndoe@domain1.net
mailStorageDirectory: maildir:/home/vmail/domain1.net/johndoe@domain1.net/Maildir
```
Once this mail record is imported into LDAP, the primary mail account including additional mail aliases defined by the `mailAlias` attribute can be verified using the [postmap](http://www.postfix.org/postmap.1.html) command.

`$ postmap -q johndoe@domain1.net ldap:/etc/postfix/ldap/ldap-vmailbox.cf`

`johndoe@domain1.net`

`$ postmap -q alias4@domain2.me ldap:/etc/postfix/ldap/ldap-aliases.cf`

`johndoe@domain1.net`

We know LDAP can find our alias because the primary `mail` account that owns the alias was returned.
