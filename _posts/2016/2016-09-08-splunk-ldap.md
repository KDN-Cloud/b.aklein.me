---
title: 'Splunk Enterprise (Free) LDAP auth in Apache'
description: Adding LDAP auth in Splunk Enterprise
date: 2017-03-08
lastmod: 2026-06-12
historical_warning_title: Historical access control warning
historical_warning_body: This workaround came from an older Splunk Free era where fronting the UI with Apache LDAP auth made sense in a homelab. Splunk licensing, auth features, and deployment patterns have changed a lot since then, so use this as reference rather than current product guidance.
historical_warning_docs_url: https://httpd.apache.org/docs/2.4/mod/mod_authnz_ldap.html
historical_warning_docs_label: Apache mod_authnz_ldap documentation
tags:
- splunk
- linux
- ldap
- openldap
- apache
- authentication
- log-management
- observability
- splunk-enterprise
- apache-auth
- apache-httpd
- mod-authnz-ldap
- reverse-proxy-auth
- access-control
- self-hosted
- sysadmin
---
{% include historical-warning.html %}

# Intro

I have used Splunk for years and still use Splunk Enterprise at work and for my own use as part of the Free license group. Back when I originally wrote this, putting Apache in front of Splunk Free was an easy way to bolt on LDAP authentication because the free tier disabled the native login flow in the way I wanted to use it.

That said, this post should be read in the right timeframe. I no longer use Apache for this kind of thing unless I have a very specific reason. It is still capable, but for lightweight reverse proxy and access-control jobs I would usually reach for Nginx or Caddy first. Both are leaner, easier to keep tidy in a small environment, and feel more natural for the kind of front-door protection this post was solving.

The Apache example below is still useful as a working reference for an older homelab pattern. It just is not the stack I would default to now.

With [Splunk Free](https://docs.splunk.com/Documentation/Splunk/6.5.5/Admin/MoreaboutSplunkFree) you had to keep your daily quota below 500 MB. Splunk Free was technically Splunk Enterprise, but with certain features disabled. In my own environment Splunk and Apache were external facing, so that meant if someone knew the URL they could simply reach the page without any kind of auth gate since Splunk Free disabled the native authentication path I wanted. The following is a block of code that can be used with Apache 2.4.

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

If I were rebuilding this idea now, I would keep the core goal the same and change the front end. Splunk behind a lighter reverse proxy, then let LDAP, SSO, or some other access layer sit in front of it cleanly. Apache got the job done. It just would not be my first pick anymore.
