---
title: 'Dovecot with LDAP'
description: How I configured Dovecot to authenticate against OpenLDAP using password lookups, what changed in modern Dovecot, and where lighter identity options fit in 2026.
date: 2016-08-08
lastmod: 2026-06-12
historical_warning_title: Historical config warning
historical_warning_body: Dovecot's configuration structure has changed significantly since this was originally written in 2016. Treat the examples below as historical reference, not copy-paste-safe production guidance for a current deployment.
historical_warning_docs_url: https://doc.dovecot.org/
historical_warning_docs_label: official Dovecot documentation
tags:
- dovecot
- ldap
- openldap
- imap
- pop3
- mail-server
- mail-authentication
- self-hosted-email
- postfix
- lmtp
- postfix-book-schema
- linux
- sysadmin
- homelab
- identity-management
- directory-services
- lldap
- authentik
- ldap-outpost
- password-lookup
- maildir
- vmail
- quota
- argon2
---

> `[ STATUS: LOG UPDATED FOR 2026 RUNTIME ENVIRONMENT ]`

{% include historical-warning.html %}

This post covers how I configured Dovecot to authenticate against OpenLDAP when I was still running my own mail stack. The Dovecot docs have always done a decent job of explaining the moving pieces, but getting a real LDAP-backed configuration working against an actual schema still took more trial and error than I wanted. This write-up exists because I got tired of pretending the interesting part was the theory.

The bigger 2026 reality is that the original approach still explains the architecture, but not all of the syntax. Dovecot changed a lot since 2016, especially around config structure and mailbox path conventions. So the value here is twofold:

- this shows how the classic OpenLDAP + Dovecot pattern was wired together
- it also explains what I would revisit immediately if I were touching it now

## Why I used password lookups

Dovecot gives you two main LDAP authentication approaches: authentication binds and password lookups.

I went with password lookups because that kept the control inside Dovecot. Dovecot can retrieve the stored password hash, compare it using its own password scheme logic, and return the rest of the mailbox metadata in the same flow. That made more sense to me than binding as the user every time and letting LDAP own the entire auth decision path.

That is still a valid design. The current Dovecot docs still support both models, but they are more explicit now about how the `passdb` and `userdb` pieces map to LDAP fields.

## The original auth switch

The first change went into `10-auth.conf`. The default usually had:

```conf
!include auth-system.conf.ext
```

enabled. I commented that out and enabled:

```conf
!include auth-ldap.conf.ext
```

instead.

That was the point where Dovecot stopped looking like "Linux box with system users" and started behaving like "mail service backed by directory lookups."

## The original passdb and userdb layout

In the older config structure, `auth-ldap.conf.ext` defined both the password and user database drivers:

```conf
passdb {
  driver = ldap
}

userdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
  default_fields = home=/home/vmail/%d/%u
}
```

The `default_fields` line built the mailbox home path from Dovecot variables. `%d` expanded to the domain and `%u` to the full username. That gave me a `domain/user` layout under `/home/vmail`, which matched how I wanted the filesystem organized at the time.

That part matters because it is exactly where modern Dovecot diverges. The old variable style and old mailbox location syntax are some of the first things I would expect to touch in a current deployment.

## The LDAP lookup file

The actual LDAP connection and lookup logic lived in `dovecot-ldap.conf.ext`:

```conf
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

There are a few important pieces here.

The `dn` and `dnpass` are the service account credentials Dovecot uses to bind to LDAP and perform lookups. I kept that account in a dedicated `System` OU instead of mixing it with user objects. That separation still makes sense.

The `base` points only at the mail subtree. I liked keeping mail data under its own OU rather than forcing Dovecot to search through a broader user tree every time.

The `pass_filter` handles authentication lookups. The `user_filter` handles mailbox/user record lookups. The two are related, but they do not have to be identical, and in practice I was glad they were separate.

## Why the schema mattered so much

The `user_attrs` line is where the schema assumptions become very real. Fields like:

- `mailHomeDirectory`
- `mailStorageDirectory`
- `mailUidNumber`
- `mailGidNumber`
- `mailQuota`

were not generic magic. They came from the mail schema I was using, including `postfix-book.schema`.

If those attributes are not present, Dovecot is not going to quietly improvise. It will simply fail to build the mailbox metadata it needs.

That is one of the things I would make much more explicit now: these posts are not "LDAP in general." They are "Dovecot and Postfix against a specific style of mail-oriented LDAP schema."

## The `mailEnabled` flag was useful

I liked the `mailEnabled=TRUE` condition in the `user_filter`:

```conf
user_filter = (&(objectClass=inetOrgPerson)(uid=%n)(mailEnabled=TRUE))
```

That gave me a clean operational switch. A user could remain in LDAP for identity purposes while their mailbox access was effectively suspended. I didn't need to delete anything or break unrelated directory data just to disable mail access.

## Password schemes are one of the places 2016 aged badly

The example above uses:

```conf
default_pass_scheme = SSHA
```

That was normal enough at the time. I would not choose it today.

If you're doing this now, use a modern password scheme such as `ARGON2I` or `PBKDF2-SHA512`, and make sure your stored LDAP password values and Dovecot's configured default scheme actually agree. Dovecot's current docs are much clearer about either setting a default scheme globally or prefixing stored hashes with the scheme.

That is one of the easiest places for a legacy config to "look fine" and still be wrong.

## Quota mapping

For quota, I used:

```conf
mailQuota=quota_rule=*:bytes=%$
```

inside `user_attrs`.

That let Dovecot pull a per-user quota straight out of LDAP if the attribute existed. If the record did not define `mailQuota`, Dovecot would fall back to whatever the global quota policy was. I still like that pattern because it keeps the default simple while allowing per-user overrides without editing Dovecot config every time.

## PAM note from the original setup

At the time I also had `/etc/pam.d/dovecot` using LDAP:

```conf
auth    required    pam_ldap.so nullok
account required    pam_ldap.so
```

That reflected how the stack was arranged then, but if I were building this cleanly today I would be very careful not to mix authentication approaches unless I had a real reason. If Dovecot is doing LDAP auth directly, I would rather keep the logic obvious than spread it across multiple layers just because Linux technically lets me.

## How I would think about this in modern Dovecot

This is the part I wish every old Dovecot post spelled out.

Modern Dovecot still supports LDAP, but the configuration model is cleaner and more explicit. Current docs talk much more directly about:

- LDAP `passdb` returning password fields
- LDAP `userdb` returning mailbox fields
- using static `userdb` when UID/GID and path patterns are predictable
- using newer variable syntax such as `%{user}` and `%{user|domain}`

That last part is a big one. In Dovecot 2.4, older mailbox location patterns like:

```conf
mail_location = maildir:/var/srv/foo/%d/%u
```

map to a newer style that looks more like:

```conf
mail_home = /home/vmail/%{user|domain}/%{user|username}
mail_driver = maildir
mail_path = ~/maildir
mail_uid = vmail
mail_gid = vmail
```

I am not saying "replace your config blindly with that." I am saying this is exactly the class of change that can break an old installation if someone assumes a 2016 post still maps 1:1 to current Dovecot syntax.

## A smarter 2026 optimization

One thing the current Dovecot docs call out more clearly than older write-ups did: if all users share the same UID/GID and your mailbox path can be templated, a static `userdb` is often better than an LDAP `userdb` lookup.

That is a good example of where modern Dovecot got more honest about operational efficiency. If the mailbox metadata is mostly deterministic, there is no point doing an extra LDAP lookup just to rediscover what you already know.

## Where this still works and where it starts to hurt

Running raw OpenLDAP for a homelab mail stack is still possible. It is also still a commitment.

Individually, none of the pieces are impossible:

- schema management
- ACLs
- replication
- indexing
- backup discipline
- auth troubleshooting

But stacked together, they become a second hobby.

If all you need is LDAP compatibility for a handful of services, [LLDAP](https://github.com/lldap/lldap) is a lot lighter. It gives you an LDAP-speaking backend without signing yourself up for the full classic OpenLDAP experience.

My own stack eventually moved toward [Authentik](https://goauthentik.io) as the primary identity layer. Authentik can expose an LDAP outpost for services that still need LDAP, which makes a lot more sense for a modern homelab than maintaining standalone directory infrastructure just because one or two services still speak LDAP.

For Dovecot specifically, that can work well, because the problem is mostly "authenticate this user and return the mailbox fields Dovecot needs." That maps more naturally to an identity platform than Postfix's mail routing tables do.

## Final take

If you're maintaining an older Dovecot + OpenLDAP stack, this post still explains the shape of the system. The architecture is recognizable. The exact syntax may not be.

If you're building fresh, I would do three things immediately:

- use modern password schemes
- verify every mailbox path and variable against current Dovecot docs
- seriously consider whether LDAP should be OpenLDAP at all, or whether a lighter identity layer makes more sense now

And if you're touching a legacy Dovecot install that has "been fine for years," do not trust muscle memory. This is one of those projects where old config habits can absolutely make a modern deployment weird in a hurry.

If you want the broader context on why I eventually stepped away from self-hosted email as a long-term lifestyle choice, that is here: [15 Years of Self-Hosted Email. Here's Why I Stopped.](/2026/06/04/self-hosted-email-migration.html)
