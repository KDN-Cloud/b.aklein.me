---
title: 'Splunk Enterprise (Free) LDAP auth in Apache'
description: Adding LDAP auth in Splunk Enterprise
date: 2017-03-08
tags:
- Splunk
- Linux
---
# Intro

I have used Splunk for years and still use Splunk Enterprise at work and for my own use as part of the Free license group. With [Splunk Free](https://docs.splunk.com/Documentation/Splunk/6.5.5/Admin/MoreaboutSplunkFree) you have to keep your daily quota below 500 MB. Splunk Free is technically Splunk Enterprise, but with certain features disabled. In my own environment Splunk and Apache are external facing, so that means if someone knows the URL they can simply login without any kind of authentication since Splunk Free disables this. The following is a block of code that can be used with Apache 2.4.

### Apache Config
Adjust `/etc/httpd/conf/extra/splunk.conf` to match your own environment as needed.

```
# LDAP auth
<proxy https://0.0.0.0:7000/*>
  Require all denied
  AuthName "This Splunk Restricted Area Requires LDAP Authentication"
  AuthType Basic
  AuthBasicProvider ldap
  AuthLDAPURL "ldap://127.0.0.1/ou=People,dc=domain,dc=net"
  Require ldap-group cn=splunk-staff,ou=Groups,dc=domain,dc=net
  AuthLDAPMaxSubGroupDepth 1
</proxy>
```
After reloading `httpd` we can see that visiting our Splunk page over SSL presents our login.
