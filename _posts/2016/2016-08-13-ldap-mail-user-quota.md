---
title: 'Setup user specific mail quotas with LDAP'
date: 2016-08-13
lastmod: 2026-06-13
description: Setting mail quotes with OpenLDAP.
historical_warning_title: Historical config warning
historical_warning_body: This quota setup came from an older OpenLDAP plus Dovecot mail stack and the broad idea still works, but modern Dovecot quota and mailbox configuration has moved around since 2016.
historical_warning_docs_url: https://doc.dovecot.org/
historical_warning_docs_label: current Dovecot docs
tags:
- dovecot
- ldap
- openldap
- quota
- self-hosted-email
- mail-server
- mail-quota
- mailbox-management
- quota-management
- imap
- maildir
- postfix-book-schema
- mail-storage
- vmail
- homelab
- linux
- sysadmin
---

{% include historical-warning.html %}

This was the pattern I used when I wanted a sane global mailbox quota in Dovecot, but still wanted the ability to override specific users directly from LDAP. That combination ended up being a lot cleaner than hardcoding exceptions in Dovecot itself.

The core idea is simple. Dovecot enforces quota, but LDAP stores the per-user override. If a user record has a `mailQuota` value, Dovecot uses it. If not, the mailbox falls back to the global default.

## The LDAP mapping

The quota logic lived in `dovecot-ldap.conf.ext` as part of the `user_attrs` mapping:

```conf
user_attrs = mailHomeDirectory=home,mailStorageDirectory=mail,mailUidNumber=uid,mailGidNumber=gid,mailQuota=quota_rule=*:bytes=%$
```

The part that matters here is:

```conf
mailQuota=quota_rule=*:bytes=%$
```

That maps the LDAP attribute `mailQuota` to a Dovecot quota rule. In practice it means Dovecot reads the raw LDAP value and turns it into a mailbox quota for that specific user.

This only works if your schema actually includes the `mailQuota` attribute. In my case that came from `postfix-book.schema`, the same schema that carried the rest of the mail-specific attributes for the OpenLDAP-backed mail stack.

## What the user record looked like

Once the mapping is in place, a user record can simply carry a quota like this:

```ldif
mailQuota: 250MB
```

That value overrides whatever global quota you have defined in Dovecot. So if the global default is 1 GB and a single user needs a smaller or larger mailbox, you do not need to touch the Dovecot config at all. You just update the LDAP record.

That was the real benefit for me. Mailbox policy stayed centralized, but individual exceptions remained easy to manage.

## The global versus per-user split

This is the part that made the whole thing worth doing.

My normal approach was:

- set one reasonable global quota in Dovecot
- only use `mailQuota` in LDAP for users who needed an override

That kept the directory cleaner. Most users inherited the default. Only the exceptions carried extra quota metadata.

If you're running a small environment, this matters more than people think. The fewer one-off edits you make to service config files, the easier it is to understand what the system is actually doing six months later.

## Restart and verification

After updating the LDAP mapping or adjusting quota settings, restart Dovecot:

```bash
systemctl restart dovecot
```

Then verify what Dovecot thinks the quota is for a specific user. I used to keep a shell alias around for this because it saved time:

```bash
alias quota='doveadm quota get -u $1'
```

Then:

```bash
quota johndoe
```

Expected output looks something like this:

```text
Quota name Type    Value  Limit                                             %
User quota STORAGE     0 256000                                             0
User quota MESSAGE     0      -                                             0
```

That output tells you the user-specific quota was actually picked up. If Dovecot is still showing the global limit instead, either the LDAP attribute is missing, the mapping is wrong, or the schema attribute name does not match what your directory actually exposes.

## What usually goes wrong

The common failure modes here were pretty predictable:

- `mailQuota` exists in the post, but not in the schema loaded into LDAP
- the LDAP mapping is correct, but the value format is wrong
- Dovecot restarted cleanly, but the wrong user record is being queried
- the per-user rule exists, but a global quota plugin config is overriding it unexpectedly

If quota behavior looks wrong, verify the LDAP record first, then the `user_attrs` mapping, then confirm with `doveadm` what Dovecot actually sees.

## Where this fits now

The broad pattern still makes sense in 2026. Per-user quota data living in the identity layer is reasonable. What changed is mostly the surrounding Dovecot configuration model and how many people are still willing to run raw OpenLDAP just to hold mail metadata.

If you're keeping an older Dovecot + OpenLDAP mail stack alive, this is still a useful pattern. If you're building fresh, I would spend more time asking whether the directory layer should still be classic OpenLDAP at all.
