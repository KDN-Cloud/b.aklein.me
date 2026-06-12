---
title: 'Using LDAP authentication with OpenVPN'
date: 2017-02-08
lastmod: 2026-06-12
description: How I configured OpenVPN to authenticate users against OpenLDAP using the openvpn-auth-ldap plugin, and where this fits in a modern homelab stack in 2026.
tags:
- openvpn
- ldap
- openldap
- vpn
- remote-access
- authentication
- linux
- sysadmin
- homelab
- self-hosted
- wireguard
- authentik
- ldap-outpost
- identity-management
- directory-services
- network-security
- pki
- tls
- infrastructure
- openvpn-auth-ldap
- vpn-authentication
- lldap
- zero-trust
---

> `[ STATUS: LOG UPDATED FOR 2026 RUNTIME ENVIRONMENT ]`

This post covers how I configured OpenVPN to authenticate users against OpenLDAP back in 2017. The setup worked well at the time and the core mechanics are still valid. OpenVPN 2.7.x is actively maintained and the LDAP plugin approach still functions. What has changed significantly is the broader context around VPN choice and identity infrastructure in a homelab, so I've added a proper 2026 update at the end.

The reason I went with LDAP authentication instead of the standard Easy-RSA PKI approach was centralized user management. With certificate-based auth you're generating and distributing a client cert for every user, revoking them manually when someone needs to be removed, and keeping track of what's been issued. With LDAP authentication you have a single directory that owns access control. Enable the user's VPN attribute and they can connect. Disable it and they can't. No certificate lifecycle to manage.

**The LDAP schema**

Before anything else you need the [openvpn-ldap schema](https://github.com/bfg/openvpn_auth/blob/master/contrib/openldap/openvpn-ldap.schema) loaded into OpenLDAP. This adds the `openVPNUser` objectClass and its associated attributes. Once it's loaded you can add VPN access to any existing People record with two lines:

```ldif
objectClass: openVPNUser
openvpnEnabled: TRUE
openvpnClientx509CN: mydomain.net
```

The `openvpnEnabled` attribute is the access gate. Set it to `TRUE` and the user can authenticate. Set it to `FALSE` and they're blocked without touching anything else in their record. This is the same pattern I used for mail accounts with `mailEnabled`: one attribute controls access, the record itself stays intact.

**Installing the plugin**

The plugin that makes OpenVPN speak to LDAP is [openvpn-auth-ldap](https://github.com/threerings/openvpn-auth-ldap) from threerings. On Debian and Ubuntu it's in the repos:

```bash
apt install openvpn-auth-ldap
```

On Arch Linux it was available as an AUR package. On other distributions you may need to build from source, which the README covers.

One thing worth knowing: the threerings repository hasn't seen active upstream development in several years. The plugin still compiles and works against current OpenVPN builds, but it's not receiving new features or security fixes. For a production environment that's worth factoring in. For a homelab it's been fine in practice.

**Server configuration**

Once the plugin is installed, add these two lines to your OpenVPN server config:

```
plugin /usr/lib/openvpn/plugins/openvpn-auth-ldap.so /etc/openvpn/auth/auth-ldap.conf
verify-client-cert none
```

Adjust the `.so` path to match where the plugin landed on your system. The `verify-client-cert none` directive tells OpenVPN to use username and password authentication rather than requiring a client certificate. If you want both, certificate plus LDAP credentials, remove that line and distribute client certs as you normally would.

**The auth-ldap.conf**

The LDAP connection and search configuration lives in a separate file. A working example looks like this:

```xml
<LDAP>
    URL             ldap://ldap.domain.net
    BindDN          uid=openvpn,ou=System,dc=domain,dc=net
    Password        yourserviceaccountpassword
    Timeout         15
    TLSEnable       no
    FollowReferrals no
</LDAP>

<Authorization>
    BaseDN          ou=People,dc=domain,dc=net
    SearchFilter    (&(uid=%u)(openvpnEnabled=TRUE))
    RequireGroup    false
</Authorization>
```

The `SearchFilter` is doing the real work here. It matches on the user's uid and checks that `openvpnEnabled` is set to `TRUE`. If either condition fails the authentication is rejected. The `BindDN` is a dedicated service account in a System OU, the same pattern used for Dovecot and Postfix LDAP service accounts. Keep service accounts separate from user accounts and give them only the permissions they actually need.

If you want to restrict VPN access to a specific LDAP group rather than using the per-user attribute approach, the `RequireGroup` and `GroupBaseDN` directives handle that. The example config in the threerings repository covers the group membership options in detail.

After editing `auth-ldap.conf`, restart OpenVPN:

```bash
systemctl restart openvpn-server@server
```

When a client connects successfully you'll see entries like this in the OpenVPN log confirming the plugin intercepted authentication and the LDAP check passed:

```
PLUGIN_CALL: POST openvpn-auth-ldap.so/PLUGIN_AUTH_USER_PASS_VERIFY status=0
TLS: Username/Password authentication succeeded for username 'tony'
```

A status of `0` means success. Anything non-zero means the plugin rejected authentication. Check your `auth-ldap.conf` bind credentials and search filter first, then verify the user record in LDAP has `openvpnEnabled: TRUE`.

**Testing the LDAP filter before touching OpenVPN**

Before restarting OpenVPN, test your search filter directly against the directory with `ldapsearch`. This catches misconfigured filters without a failed VPN connection to debug:

```bash
ldapsearch -x -H ldap://ldap.domain.net \
  -D "uid=openvpn,ou=System,dc=domain,dc=net" \
  -w yourpassword \
  -b "ou=People,dc=domain,dc=net" \
  "(&(uid=tony)(openvpnEnabled=TRUE))"
```

If that returns the user record, OpenVPN will be able to authenticate that user. If it returns nothing, the filter or the base DN is wrong.

**Where this sits in 2026**

OpenVPN is still actively maintained, version 2.7.4 shipped in April 2026, and this setup still works. But the honest answer is that if I were building this from scratch today I would use WireGuard instead, and I have. WireGuard is built into the Linux kernel, has a dramatically simpler configuration surface, and is significantly faster. The peer-based model is different from OpenVPN's client-server model but for homelab remote access it's actually more appropriate. I wrote about the full WireGuard journey separately, including the site-to-site setup to a couple of Vultr VPS nodes.

For LDAP-authenticated VPN access in a modern homelab, [NetBird](https://netbird.io) is worth looking at. It's WireGuard-based, open source, self-hostable, and integrates natively with identity providers including Authentik, Keycloak, and Azure AD. You get SSO-controlled VPN access with a management dashboard and proper audit logging, which is the modern equivalent of what this openvpn-auth-ldap setup was doing in 2017, just with less operational overhead and a better security model.

If you're already running Authentik as your identity provider, you can also use its LDAP outpost to back this exact OpenVPN setup instead of raw OpenLDAP. The connection strings in `auth-ldap.conf` would point at the Authentik LDAP outpost instead, and user access gets controlled from Authentik's interface. That's a reasonable upgrade path if you want to keep OpenVPN but modernize the identity layer behind it.
