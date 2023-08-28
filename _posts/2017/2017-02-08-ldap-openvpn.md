---
title: 'Using LDAP authentication with OpenVPN'
date: 2017-02-08
---

# Intro
With OpenVPN it is quite common to use Easy-RSA to create a Public Key Infrastructure (PKI) so that client certificates may be distributed. For my use case I much prefer to use LDAP authentication with OpenVPN. I use OpenLDAP but any LDAP server should be fine. I am also using an Arch [PKGBUILD](https://aur.archlinux.org/packages/openvpn-auth-ldap/) file to build the actual plugin that makes OpenVPN work with LDAP auth. 

### LDAP Prerequisite
Before anything can work we need to have an [OpenVPN LDAP schema](https://github.com/bfg/openvpn_auth/blob/master/contrib/openldap/openvpn-ldap.schema) loaded into our environment. While this LDAP schema offers many attributes, for my use case I only care about having authorized VPN users connect. Once `openvpn-ldap.schema` is loaded, an LDAP record can contain a new VPN objectClass and attributes. 

```
...
objectClass: openVPNUser
openvpnEnabled: TRUE
openvpnClientx509CN: mydomain.net
...
```
The nice thing about this is we can easily modify a `People` record to enable or disable VPN user access.

### LDAP OpenVPN Config
There are only a few fairly simple things to do once our environment is ready for OpenVPN.

1. Build and install the [openvpn-auth-ldap](https://github.com/threerings/openvpn-auth-ldap) plugin. On Arch Linux you can easily [build and install the plugin from AUR](https://aur.archlinux.org/packages/openvpn-auth-ldap/).
2. Add the following to your OpenVPN server configuration:
  ```
  plugin /usr/lib/openvpn/plugins/openvpn-auth-ldap.so /etc/openvpn/auth/auth-ldap.conf
  ```
Adjust the paths for `openvpn-auth-ldap.so` and `auth-ldap.conf` as needed.
3. It is a good idea to keep a default copy of `auth-ldap.conf`. An example configuration can be found on [GitHub](https://github.com/threerings/openvpn-auth-ldap/wiki/Configuration). Since I like to be organized I keep my LDAP config inside `/etc/openvpn/auth`. 
4. Restart OpenVPN after modifying `auth-ldap.conf` accordingly. With systemd one can execute `systemctl restart openvpn-server@server`, respectively. 

---
**INFO**

_Upon restarting, `openvpn.log` should show a plugin initialization entry similar to_
```
PLUGIN_INIT: POST /usr/lib/openvpn/plugins/openvpn-auth-ldap.so '[/usr/lib/openvpn/plugins/openvpn-auth-ldap.so] [/etc/openvpn/auth/auth-ldap.conf]' intercepted=PLUGIN_AUTH_USER_PASS_VERIFY|PLUGIN_CLIENT_CONNECT|PLUGIN_CLIENT_DISCONNECT
```
---

At this point we can now have our VPN client authenticate with a username and password using our LDAP auth backend. We can see successful LDAP connections in `openvpn.log` when a new client connects.

```
127.0.0.1:60923 TLS: Initial packet from [AF_INET]127.0.0.1:60923, sid=42bec808 6635b5f5
127.0.0.1:60923 peer info: IV_VER=2.4.3
127.0.0.1:60923 peer info: IV_PLAT=mac
127.0.0.1:60923 peer info: IV_PROTO=2
127.0.0.1:60923 peer info: IV_NCP=2
127.0.0.1:60923 peer info: IV_LZ4=1
127.0.0.1:60923 peer info: IV_LZ4v2=1
127.0.0.1:60923 peer info: IV_LZO=1
127.0.0.1:60923 peer info: IV_COMP_STUB=1
127.0.0.1:60923 peer info: IV_COMP_STUBv2=1
127.0.0.1:60923 peer info: IV_TCPNL=1
127.0.0.1:60923 PLUGIN_CALL: POST /usr/lib/openvpn/plugins/openvpn-auth-ldap.so/PLUGIN_AUTH_USER_PASS_VERIFY status=0
127.0.0.1:60923 TLS: Username/Password authentication succeeded for username 'tony'
127.0.0.1:60923 Control Channel: TLSv1.2, cipher TLSv1.2 DHE-RSA-AES256-GCM-SHA384
127.0.0.1:60923 [] Peer Connection Initiated with [AF_INET]127.0.0.1:60923
127.0.0.1:60923 MULTI_sva: pool returned IPv4=10.8.0.6, IPv6=(Not enabled)
127.0.0.1:60923 PLUGIN_CALL: POST /usr/lib/openvpn/plugins/openvpn-auth-ldap.so/PLUGIN_CLIENT_CONNECT status=0
127.0.0.1:60923 OPTIONS IMPORT: reading client specific options from: /tmp/openvpn_cc_0e9bde0ccb231123af86cd70e8a6f37c.tmp
127.0.0.1:60923 MULTI: Learn: 10.8.0.6 -> 127.0.0.1:60923
127.0.0.1:60923 MULTI: primary virtual IP for 127.0.0.1:60923: 10.8.0.6
127.0.0.1:60923 PUSH: Received control message: 'PUSH_REQUEST'
127.0.0.1:60923 SENT CONTROL [UNDEF]: 'PUSH_REPLY,route 192.168.128.0 255.255.128.0,route 10.8.0.0 255.255.255.0,redirect-gateway def1 bypass-dhcp,dhcp-option DNS 208.67.222.222,dhcp-option DNS 208.67.220.220,route 10.8.0.0 255.255.255.0,topology net30,ping 10,ping-restart 120,ifconfig 10.8.0.6 10.8.0.5,peer-id 1,cipher AES-256-GCM' (status=1)
127.0.0.1:60923 Data Channel: using negotiated cipher 'AES-256-GCM'
127.0.0.1:60923 Data Channel Encrypt: Cipher 'AES-256-GCM' initialized with 256 bit key
127.0.0.1:60923 Data Channel Decrypt: Cipher 'AES-256-GCM' initialized with 256 bit key
```

