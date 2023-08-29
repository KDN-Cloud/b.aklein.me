---
title: 'Setup user specific mail quotas with LDAP'
date: 2016-08-13
description: Setting mail quotes with OpenLDAP.
tags:
- Mail
- LDAP
- Quota
---

# Intro
The official [Dovecot wiki](http://wiki2.dovecot.org/Quota) should be your go to for setting up mail quotas, but here I am describing how I setup mail-user specific quotas to work with my LDAP environment.

### Setup
I included a quota configuration for `user_attrs` in my `dovecot-ldap.conf.ext` consisting of the following

```
user_attrs = mailHomeDirectory=home,mailStorageDirectory=mail,mailUidNumber=uid,mailGidNumber=gid,mailQuota=quota_rule=*:bytes=%$
```

The quota limit is in the **mailQuota** field: `mailQuota=quota_rule=*:bytes=%$`

Once Dovecot has been restarted with the above quota limit, we can then add the `mailQuota` attribute with a value using a preferred metric unit. For example, a mail user record might have a quota limit of 250 MB.

`mailQuota: 250MB`

The above quota is user-specific so this will end up overriding the [global quota](http://wiki2.dovecot.org/Quota/Configuration).

### Verify Quota

I use a lot of aliases to save time, so putting this in your user profile is recommended.

`alias quota='doveadm quota get -u $1 '`

```
$ quota johndoe
Quota name Type    Value  Limit                                             %
User quota STORAGE     0 256000                                             0
User quota MESSAGE     0      -                                             0
```
See the [doveadm-quota wiki](http://wiki2.dovecot.org/Tools/Doveadm/Quota) for additional options.