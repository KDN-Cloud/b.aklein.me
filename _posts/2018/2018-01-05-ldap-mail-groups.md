---
title: 'LDAP Mail Distribution Groups with Postfix'
description: Setup LDAP mail groups with Postfix
date: 2018-01-05
lastmod: 2026-06-13
historical_warning_title: Historical group routing warning
historical_warning_body: This post documents LDAP-backed distribution groups in a classic Postfix and OpenLDAP mail stack. The model still works, but it assumes the older schema-driven self-hosted mail environment I was running at the time.
historical_warning_docs_url: https://www.postfix.org/LDAP_README.html
historical_warning_docs_label: official Postfix LDAP documentation
tags:
- postfix
- ldap
- linux
- mail-groups
- openldap
- distribution-lists
- self-hosted-email
- mail-server
- group-mail
- mail-aliases
- virtual-alias-maps
- postfix-book-schema
- mail-routing
- homelab
- sysadmin
---

{% include historical-warning.html %}

A common feature in any real mail environment is the distribution group. One address, multiple recipients, all driven from one place. In my case I wanted that one place to be LDAP.

The use case is obvious enough: `hq@example.net`, `staff@example.net`, `billing@example.net`, whatever address represents a team rather than a person. Instead of managing those expansions in flat files, I wanted the group membership stored directly in the same directory layer as the rest of the mail environment.

## Postfix
The first step is telling Postfix where the LDAP group lookup lives.

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
The group attributes come from [postfix-book.schema](https://github.com/variablenix/ldap-mail-schema/blob/master/postfix-book.schema), so that schema needs to be loaded first or the directory will reject the record structure.

## What the group record looked like

An LDAP mail distribution group could look like this:

```ldif
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

Now when a message is sent to `hq@example`, Postfix expands the LDAP lookup and delivers to every address defined by `mailGroupMember`.

That is the whole pattern. One mail address in LDAP, one list of destination members, and Postfix doing the expansion during alias resolution.

## Why I liked this model

The nice part was operational simplicity.

- add a user to the group by adding another `mailGroupMember`
- remove a user by deleting one attribute value
- manage the whole thing through LDAP tools you are already using

If you are already committed to a directory-driven mail stack, this fits naturally.

## How to test it

Once the lookup is configured, do not just assume it works. Test it.

Start by querying the table directly:

```bash
postmap -q hq@example ldap:/etc/postfix/ldap/ldap-groups.cf
```

If Postfix resolves the group correctly, you should see one of the member addresses returned. Depending on how the lookup is consumed, Postfix will then continue processing the remaining results as part of alias expansion.

Then send a real test message and tail the mail log:

```bash
tail -f /var/log/mail.log
```

If everything is wired correctly, the log will show the message being expanded and delivered to all group members.

## Where this fits now

This pattern still makes sense if you are already running the old OpenLDAP mail schema. It is one of the cleaner examples of LDAP being used for something directory-shaped rather than forcing it into a job it never really wanted.

That said, it is still part of the bigger question I keep coming back to with these old posts: the design is valid, but do you still want to be the person maintaining the directory, the schema, the lookup tables, the mail flow, and all the little edge cases that come with it?
