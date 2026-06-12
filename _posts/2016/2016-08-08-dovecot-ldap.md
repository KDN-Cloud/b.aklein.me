---
title: 'Dovecot with LDAP'
description: How I configured Dovecot to authenticate against OpenLDAP using password lookups, and how that setup maps to modern identity providers like Authentik in 2026.
date: 2016-08-08
lastmod: 2026-06-12
tags:
- dovecot
- ldap
- openldap
- authentik
- ldap-outpost
- imap
- mail-server
- mail-authentication
- self-hosted-email
- postfix
- postfix-book-schema
- linux
- sysadmin
- homelab
- identity-management
- directory-services
- lldap
- password-lookup
- vmail
- maildir
- quota
- pam
- self-hosted
- infrastructure
---

> `[ STATUS: LOG UPDATED FOR 2026 RUNTIME ENVIRONMENT ]`

This post covers how I configured Dovecot to authenticate against OpenLDAP when I was running my own mail stack. The Dovecot wiki does a solid job explaining the options at a high level, but the actual configuration details for getting password lookups working with specific LDAP attributes took more trial and error than I expected. That's what this documents.

Before getting into the config: if you're setting this up fresh in 2026, the OpenLDAP path is worth evaluating carefully. Raw OpenLDAP is operationally heavy and the self-hosting community has largely moved toward lighter alternatives. I'll cover the modern options at the end of this post. The config below is accurate and still works, but it's worth knowing where the ecosystem has gone since 2016.

Dovecot offers two approaches to LDAP authentication: authentication binds and password lookups. I went with password lookups because it's the recommended approach. With auth binds, Dovecot connects to LDAP as the user being authenticated, which means it needs to handle failed bind attempts and you lose the ability to use Dovecot's own password scheme handling. Password lookups keep that control inside Dovecot where it belongs.

The first change goes in `10-auth.conf`. The default has `!include auth-system.conf.ext` enabled. Comment that out and uncomment `!include auth-ldap.conf.ext` instead.

`auth-ldap.conf.ext` then needs to define both the password and user database drivers:

```
passdb {
  driver = ldap
}

userdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
  default_fields = home=/home/vmail/%d/%u
}
```

The `default_fields` line sets the mailbox home path using Dovecot's variable substitution. `%d` expands to the mail domain and `%u` to the full username, so you end up with a `domain/user` directory structure under `/home/vmail`. This matches how I organized the vmail storage and it keeps things easy to navigate on the filesystem.

The actual LDAP connection and lookup configuration lives in `dovecot-ldap.conf.ext`:

```
hosts = ldap.domain.net ldap.domain2.net ldap.domain3.net
auth_bind = no
dn = uid=dovecot,ou=System,dc=domain,dc=net
dnpass = MyP@sswd
ldap_version = 3
base = ou=Mail,dc=domain,dc=net
deref = never
scope = subtree
default_pass_scheme = SSHA

# user filter
user_attrs = mailHomeDirectory=home,mailStorageDirectory=mail,mailUidNumber=uid,mailGidNumber=gid,mailQuota=quota_rule=*:bytes=%$
user_filter = (&(objectClass=inetOrgPerson)(uid=%n)(mailEnabled=TRUE))

# password filter
pass_attrs = mail=user,userPassword=password
pass_filter = (&(objectClass=inetOrgPerson)(uid=%n))

iterate_attrs = mail=user
iterate_filter = (objectClass=inetOrgPerson)
```

A few things worth explaining here. The `dn` and `dnpass` are the credentials for the service account Dovecot uses to bind to LDAP and perform lookups. I kept this account in a dedicated `System` OU rather than mixing it with regular users. The `base` points to the `Mail` OU specifically since that's where mail user records live, separate from the `People` OU that holds general directory users.

The `user_attrs` mapping is where the postfix-book.schema attributes come in. `mailHomeDirectory`, `mailStorageDirectory`, `mailUidNumber`, `mailGidNumber`, and `mailQuota` are all custom attributes defined in that schema. If you're using this configuration, you need the [postfix-book.schema](https://github.com/variablenix/ldap-mail-schema/blob/master/postfix-book.schema) loaded into OpenLDAP or these attribute lookups will return nothing and Dovecot will refuse authentication.

The `mailEnabled=TRUE` check in `user_filter` is worth calling out specifically. It means a user record can exist in LDAP without being able to receive mail. Setting that attribute to `FALSE` on a record effectively suspends the mailbox without deleting anything, which was useful for managing accounts cleanly.

On the `default_pass_scheme` value: the config above uses `SSHA`, which was standard practice in 2016. It's not what you should use now. If you're setting this up today, use `ARGON2I` or `PBKDF2-SHA512`. Update both the `default_pass_scheme` value in Dovecot and the password generation for your LDAP records accordingly.

For quota, the `mailQuota=quota_rule=*:bytes=%$` mapping in `user_attrs` pulls a per-user quota from the LDAP record if one exists. If no `mailQuota` attribute is present on the record, Dovecot falls back to whatever global quota you have defined in `dovecot.conf`. That combination gives you a sane default with the ability to override per user without touching the Dovecot config.

The last piece on the PAM side: `/etc/pam.d/dovecot` needs to tell PAM to use LDAP for authentication:

```
auth    required    pam_ldap.so nullok
account required    pam_ldap.so
```

Once everything is in place, restart Dovecot and watch the mail log for errors:

```bash
systemctl restart dovecot
tail -f /var/log/mail.log
```

Any misconfiguration in the LDAP filter or attribute mapping will show up immediately in the log as authentication failures. The most common issues are a wrong `base` DN, an incorrect service account bind, or a missing schema attribute.

**Where this fits in 2026**

Running raw OpenLDAP for a homelab mail stack is a significant operational commitment. Schema management, replication, access control lists, index tuning — none of it is hard individually but it adds up quickly. If you're building a new setup and don't already have OpenLDAP running for other reasons, it's worth considering lighter alternatives.

[LLDAP](https://github.com/lldap/lldap) is a minimal LDAP server built specifically for self-hosting scenarios. It speaks the LDAP protocol for services that need it, exposes a clean web UI for user management, and requires almost no configuration to get running. If all you need is an LDAP backend for mail authentication, LLDAP is considerably less overhead than a full OpenLDAP deployment.

My own stack has moved to [Authentik](https://goauthentik.io) as the primary identity provider. Authentik handles SSO, OAuth2, SAML, and exposes an LDAP outpost for services that need to authenticate via LDAP directly. Dovecot can authenticate against the Authentik LDAP outpost the same way it would against OpenLDAP, with the connection string pointing at the outpost instead. The practical difference is that you get a single identity layer managing everything, a proper web UI, full audit logging, and no schema files to maintain. For a homelab running more than just mail, it makes a lot more sense than keeping a separate OpenLDAP instance alive.

If you want the full context on why I eventually moved off self-hosted email entirely, that's covered here: [15 Years of Self-Hosted Email. Here's Why I Stopped.](/2026/06/04/self-hosted-email-migration.html)
