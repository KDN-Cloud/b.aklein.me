---
title: 'Adding official email accounts and aliases in LDAP'
description: How I structured OpenLDAP mail user records with the PostfixBookMailAccount schema, keeping People and Mail containers separate and verifying with postmap.
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
---

> `[ STATUS: LOG UPDATED FOR 2026 RUNTIME ENVIRONMENT ]`

This post covers how I structured OpenLDAP mail user records when I was running my own Postfix and Dovecot stack. The specific schema and objectClass setup here is worth documenting because the separation between People and Mail containers is something I landed on after trying a few different approaches and this one made the most sense operationally.

The idea is simple. The `People` organizational unit holds standard user records for authentication and general directory purposes. The `Mail` OU holds records that are purely mail-specific, with attributes that have no business living on a general user account. Keeping them separate means you can manage mail accounts independently, add aliases without touching the main user record, and disable a mailbox without affecting anything else in the directory.

For the mail-specific attributes to work you need the [postfix-book.schema](https://github.com/variablenix/ldap-mail-schema/blob/master/postfix-book.schema) loaded into OpenLDAP. This schema defines the `PostfixBookMailAccount` objectClass along with attributes like `mailEnabled`, `mailAlias`, `mailUidNumber`, `mailGidNumber`, `mailHomeDirectory`, and `mailStorageDirectory`. Without it LDAP will reject any record that tries to use those attributes.

Here is what a mail account record looks like in LDIF format:

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
userPassword: {SSHA}lFXu8SajJaj+vEk99SvsBa+sRLmLfiRV
mailHomeDirectory: /home/vmail/domain1.net/johndoe@domain1.net
mailStorageDirectory: maildir:/home/vmail/domain1.net/johndoe@domain1.net/Maildir
```

A few things worth pointing out in this record. The `mailEnabled` attribute is what Postfix checks to determine whether to accept mail for the account. Setting it to `FALSE` effectively disables delivery without removing the record from the directory. The `mailAlias` attribute is multi-valued so you can stack as many aliases as you need on a single account, including across multiple domains. The `mailUidNumber` and `mailGidNumber` map to the system user that owns the vmail directory, which is typically a dedicated `vmail` user rather than a real system account.

The `mailStorageDirectory` attribute defines where Dovecot will store the mailbox. The `maildir:` prefix tells Dovecot to use Maildir format rather than mbox, which is what you want for any serious deployment since Maildir stores each message as a separate file and handles concurrent access cleanly.

Once the record is imported into LDAP you can verify that Postfix can look it up correctly using `postmap` in query mode. This tests your LDAP lookup configuration against the live directory without touching mail flow:

```bash
$ postmap -q johndoe@domain1.net ldap:/etc/postfix/ldap/ldap-vmailbox.cf
johndoe@domain1.net

$ postmap -q alias4@domain2.me ldap:/etc/postfix/ldap/ldap-aliases.cf
johndoe@domain1.net
```

The first query confirms the primary mailbox is resolvable. The second confirms alias lookup is working correctly. When you query an alias, Postfix returns the primary mail address that owns it, which is the expected behavior. If either of these returns nothing, the issue is almost always in the LDAP filter defined in your `ldap-vmailbox.cf` or `ldap-aliases.cf` configuration file rather than in the directory record itself.

This structure held up well across the years I ran it. The clean separation between People and Mail records made it easy to provision new mailboxes, suspend accounts, and manage aliases without touching anything outside the Mail OU.

One thing worth flagging if you're using this record as a reference: the `{SSHA}` password hash in the example above is what I was using in 2016 and it was standard practice at the time. It's not what you should be using now. SSHA is considered weak by current standards and both OpenLDAP and Dovecot support stronger schemes. If you're setting this up fresh, use `{ARGON2I}` or `{PBKDF2-SHA512}` depending on what your OpenLDAP build supports. You can generate an Argon2 hash with `slappasswd -h {ARGON2I}` on a system with the argon2 module loaded. The rest of the record structure stays exactly the same.

On the broader identity stack question: if you're standing up a fresh homelab in 2026 and haven't committed to raw OpenLDAP yet, it's worth considering whether you actually need it. OpenLDAP is powerful and flexible but it demands real operational investment, especially around replication, schema management, and access control. For a homelab mail stack specifically, the overhead of maintaining a full LDAP directory is hard to justify unless you're also using it for SSH key auth, VPN authentication, or other services that genuinely benefit from a centralized directory.

My own stack evolved significantly since this post. I moved to [Authentik](https://goauthentik.io) as my primary identity provider, which handles SSO, OAuth2, SAML, and LDAP outpost functionality from a single interface. For services that need LDAP specifically, Authentik exposes an LDAP outpost that speaks the protocol without you having to run and maintain raw OpenLDAP. It fits better into how I think about infrastructure now: fewer moving parts, more visibility, easier to reason about when something goes wrong. The web UI alone is worth it compared to managing schema files and ACLs by hand.

If you want the full context on why I eventually moved off self-hosted email entirely, including the operational reality of running Postfix and Dovecot long-term, I wrote about that here: [15 Years of Self-Hosted Email. Here's Why I Stopped.](/2026/06/04/self-hosted-email-migration.html)
