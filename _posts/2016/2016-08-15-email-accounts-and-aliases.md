---
title: 'Adding official email accounts and aliases in LDAP'
description: How I structured OpenLDAP mail user records with the PostfixBookMailAccount schema — from a separated People and Mail OU design to a single unified People record, and why the merged approach won out.
date: 2016-08-15
lastmod: 2026-06-12
tags:
- openldap
- ldap
- postfix
- dovecot
- mail-server
- self-hosted-email
- email-aliases
- identity-management
- linux
- sysadmin
- postfix-book-schema
- inetorgperson
- vmail
- maildir
- homelab
- infrastructure
- directory-services
- authentik
- sso
- oauth2
- ldap-outpost
- argon2
- password-hashing
- ldap-schema
- objectclass
- mail-routing
---

> `[ STATUS: LOG UPDATED FOR 2026 RUNTIME ENVIRONMENT ]`

This post covers how I structured OpenLDAP mail user records when I was running my own Postfix and Dovecot stack. What started as a clean separation between People and Mail organizational units eventually collapsed into a single unified record design, and both approaches are worth documenting because the reasoning behind each one is genuinely useful if you're thinking through the same problem.

For any of this to work you need the [postfix-book.schema](https://github.com/variablenix/ldap-mail-schema/blob/master/postfix-book.schema) loaded into OpenLDAP. This schema defines the `PostfixBookMailAccount` objectClass and the mail-specific attributes that come with it: `mailEnabled`, `mailAlias`, `mailUidNumber`, `mailGidNumber`, `mailHomeDirectory`, `mailStorageDirectory`, and `mailQuota`. Without it LDAP will reject any record that tries to use those attributes.

**Approach 1: Separated People and Mail OUs**

The original design kept two distinct organizational units. The `People` OU held standard user records for authentication and general directory lookups. The `Mail` OU held a completely separate set of records with mail-specific attributes that had no business living alongside a general user account.

The logic was clean on paper. You could provision a mail account without touching the People record. You could disable a mailbox by flipping `mailEnabled` to `FALSE` on the Mail record without affecting anything else in the directory. Aliases could be added and removed without any risk of changing authentication credentials. The two concerns were fully decoupled.

A Mail OU record looked like this:

```ldif
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
userPassword: {ARGON2I}...
mailHomeDirectory: /home/vmail/domain1.net/johndoe@domain1.net
mailStorageDirectory: maildir:/home/vmail/domain1.net/johndoe@domain1.net/Maildir
```

In practice the separation created its own problems. Every user existed twice in the directory. Provisioning a new account meant creating two records, keeping them in sync, and making sure any tooling that touched one was aware of the other. When a user changed their password, you had to decide which record owned it — and if both records needed a password for different services, you now had two credentials to manage for one person. The clean abstraction started leaking the moment real operational work touched it.

**Approach 2: Unified People records**

Eventually I moved everything into a single record under the `People` OU. The `PostfixBookMailAccount` objectClass got added directly to the standard user record alongside `inetOrgPerson`, merging all the mail-specific attributes into the same entry that already handled authentication.

A unified People record looks like this:

```ldif
dn: uid=jdoe,ou=People,dc=domain1,dc=net
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: PostfixBookMailAccount
uid: jdoe
cn: John Doe
sn: Doe
mail: johndoe@domain1.net
mailEnabled: TRUE
mailAlias: alias1@domain1.net
mailAlias: alias2@domain2.net
mailAlias: alias3@domain1.net
mailAlias: alias4@domain2.net
mailUidNumber: 5000
mailGidNumber: 5000
mailHomeDirectory: /home/vmail/domain1.net/johndoe@domain1.net
mailStorageDirectory: maildir:/home/vmail/domain1.net/johndoe@domain1.net/Maildir
mailQuota: 2147483648
userPassword: {ARGON2I}...
loginShell: /bin/bash
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/jdoe
description: John Doe
```

Everything lives in one place. Authentication, mail delivery, alias lookup, quota — all driven from a single LDAP record. Provisioning a new user means creating one entry, not two. Password changes happen in one place. LDAP searches that need to cross-reference mail and identity attributes don't have to query two different OUs and join the results. The directory reflects reality: one person, one record.

The `mailEnabled` attribute still does its job here. Flipping it to `FALSE` disables mail delivery for the account without removing any attributes or touching the authentication side. The user can still log in to other LDAP-aware services, they just won't receive mail until you re-enable it. That operational separation you wanted from the two-OU design still exists, it's just handled by a single attribute on a single record rather than by the physical separation of entries.

The `mailAlias` attribute is multi-valued, so you can stack as many aliases as you need across multiple domains on the same record. Each additional alias is just another line — no structural change required.

**Verifying with postmap**

Once a record is in the directory you can verify that Postfix can resolve it correctly using `postmap` in query mode. This hits your LDAP lookup configuration directly without touching live mail flow:

```bash
$ postmap -q johndoe@domain1.net ldap:/etc/postfix/ldap/ldap-vmailbox.cf
johndoe@domain1.net

$ postmap -q alias4@domain2.net ldap:/etc/postfix/ldap/ldap-aliases.cf
johndoe@domain1.net
```

The first query confirms the primary mailbox resolves. The second confirms alias lookup is working correctly — Postfix returns the primary mail address that owns the alias, which is the expected behavior. If either returns nothing, the problem is almost always in the LDAP filter string inside your `ldap-vmailbox.cf` or `ldap-aliases.cf` configuration file rather than in the directory record itself. Check the `query_filter` and `result_attribute` values in those files and make sure the base DN matches where your records actually live.

**On the password hash**

Both example records above use `{ARGON2I}` as the password scheme placeholder. The original 2016 post used `{SSHA}`, which was standard practice at the time but is now considered weak. If you're setting this up today use `{ARGON2I}` or `{PBKDF2-SHA512}` depending on what your OpenLDAP build has compiled in. Generate an Argon2 hash with:

```bash
slappasswd -h {ARGON2I}
```

You'll be prompted for the password and it will output the full hashed value ready to paste into your LDIF.

**Where this sits in 2026**

The unified People record approach is more practical to operate than the separated design, but the honest answer is that raw OpenLDAP carries significant overhead regardless of how you structure it. Schema management, replication, access control, and index tuning all demand ongoing attention. For a homelab that's primarily running mail and nothing else, it's hard to justify.

My own stack moved to [Authentik](https://goauthentik.io) as the primary identity layer. It handles SSO, OAuth2, SAML, and exposes an LDAP outpost for services that need to authenticate via LDAP directly. The web UI alone removes most of the operational overhead that raw OpenLDAP demands, and having a single identity system managing everything is considerably easier to reason about than keeping a separate LDAP instance alive alongside other auth infrastructure.

If you want the full context on why I eventually moved off self-hosted email entirely, that's covered here: [15 Years of Self-Hosted Email. Here's Why I Stopped.](/2026/06/04/self-hosted-email-migration.html)
