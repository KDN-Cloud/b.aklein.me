---
title: 'LDAP Mail Distribution Groups with Postfix'
description: Setup LDAP mail groups with Postfix
date: 2018-01-05
tags:
- Postfix
- LDAP
- Linux
- Email
- MailGroups
---

A common feature with mail environments is to use distribution groups that you could add and remove group members from. This is fairly common among organizations. For example, one might have `hq@example.net` and a list of members stored in LDAP. I wanted to have the ability to use mail distribution groups with my OpenLDAP infrastructure. LDAP group members could then easily be removed or added using `ldapmodify` or Apache Directory Studio.

## Postfix
First, we need to tell Postfix about our LDAP distribution group config.

1. Open `/etc/postfix/main.cf`
2. Edit the `virtual_alias_maps` line and put `ldap:/etc/postfix/ldap/ldap-groups.cf` _after_ the aliases definition.
```
virtual_alias_maps = ldap:/etc/postfix/ldap/ldap-aliases.cf,ldap:/etc/postfix/ldap/ldap-groups.cf
```
Since the LDAP server is local I do not need TLS in `ldap-groups.cf`. The following is sufficient.

### ldap-groups.cf
```
server_host = ldap://localhost
search_base = ou=Groups,ou=Mail,dc=example,dc=net
version = 3
bind = no
query_filter = mail=%s
result_attribute = mailGroupMember
```
!!!! The group attrributes can be loaded using [postfix-book schema](https://github.com/variablenix/ldap-mail-schema/blob/master/postfix-book.schema)

An LDAP mail distribution group could look like this.

```
dn: mail=hq@example.email,ou=Groups,ou=Mail,dc=example,dc=net
objectClass: top
objectClass: organizationalPerson
objectClass: PostfixBookMailAccount
mail: hq@example
mailEnabled: TRUE
mailUidNumber: 5000
mailGidNumber: 5000
cn: hq
sn: group
description: hq@example distribution group
mailGroupMember: user1@example
mailGroupMember: user2@example
mailGroupMember: user3@example
mailGroupMember: user4@example
mailGroupMember: user5@example
mailGroupMember: user6@example
```
So now, when an email is sent to `hq@example` that email will land in every group member's Inbox. Each group member will be defined by the `mailGroupMember` attribute.

Once you have this configured it is a good idea to `tail` the logs and send a test mail to the group. If everything is setup correctly the mail logs will show the email delivered to all group members.