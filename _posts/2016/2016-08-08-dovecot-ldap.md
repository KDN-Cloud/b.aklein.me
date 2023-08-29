---
title: 'Dovecot with LDAP'
description: Setting up Dovecot with OpenLDAP.
date: 2016-08-08
tags:
- LDAP
- Linux
- Mail
- Dovecot
---

# Intro
The [Dovecot wiki](https://doc.dovecot.org/configuration_manual/howto/?highlight=ldap) does a really good job at explaining how to have Dovecot and OpenLDAP work together, but in this post I will describe the steps I took to configure Dovecot to work with OpenLDAP on a Linux host.

### LDAP authentication
As described in the wiki - Dovecot offers two ways to perform LDAP authentication, but I chose [LDAP password lookups](https://doc.dovecot.org/configuration_manual/authentication/ldap_passwords/). This is recommended over authentication binds.

#### 10-auth.conf
Normally `!include auth-system.conf.ext` is enabled, but this should be commented and `!include auth-ldap.conf.ext` uncommented.

#### auth-ldap.conf.ext
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
Here we're simply telling Dovecot to use LDAP instead of PAM or MySQL, respectively. For `default_fields` I'm using a `domain/user` structure as referenced by the **%d** and **%u** [variables](https://doc.dovecot.org/configuration_manual/config_file/config_variables/?highlight=variables) you can pass to Dovecot. Following this was configuring the relevant options in `dovecot-ldap.conf.ext`. 

#### dovecot-ldap.conf.ext
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
pass_attrs  = mail=user,userPassword=password
pass_filter = (&(objectClass=inetOrgPerson)(uid=%n))

iterate_attrs = mail=user
iterate_filter = (objectClass=inetOrgPerson)
```

Because I am using specific LDAP attributes shown in both `user_attrs` and `user_filter` I needed to get [postfix-book.schema](https://github.com/variablenix/ldap-mail-schema/blob/master/postfix-book.schema) loaded into OpenLDAP. 

#### Quota
While I use a global quota I also like the option of setting user specific quotas. Since I'm using postfix-book.schema in OpenLDAP, `mailQuota=quota_rule=*:bytes=%$` works just fine so that the `mailQuota` attribute can be added to mail user records.

#### dovecot.conf 
#### PAM
One last thing I needed to do was tell PAM that Dovecot should use LDAP for authentication. This involved editing `/etc/pam.d/dovecot` with the following
```
auth    required        pam_ldap.so nullok
account required        pam_ldap.so
```

### Final
Once everything has been verified the last thing is to restart Dovecot. With systemd one can execute `systemctl restart dovecot`. It's also a good idea to verify no errors are shown in the mail log using`tail -f /path/to/mail.log`. 