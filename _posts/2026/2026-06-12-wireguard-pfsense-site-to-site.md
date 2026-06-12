---
title: 'WireGuard on pfSense: Remote Access, VLANs, and Site-to-Site to Vultr'
date: 2026-06-12
description: How I replaced OpenVPN with WireGuard on my Netgate 6100, set up split-tunnel and full-tunnel peer templates across VLANs, and extended it to site-to-site tunnels with two Vultr VPS nodes running Ubuntu, automated end-to-end with Ansible.
tags:
- wireguard
- pfsense
- pfsense-plus
- netgate
- netgate-6100
- vpn
- site-to-site
- remote-access
- homelab
- self-hosted
- linux
- ubuntu
- debian
- vultr
- vps
- ansible
- ansible-playbook
- network-security
- infrastructure
- vpn-server
- vpn-tunnel
- vpn-configuration
- split-tunnel
- full-tunnel
- vlan
- firewall
- firewall-rules
- routing
- static-routes
- wireguard-linux
- wireguard-pfsense
- gitops
- proxmox
- kdn-lab
- udp
- peer-config
- qr-code
- wg-quick
- systemd
- ops-automation
---

I ran OpenVPN for years. I had it wired into LDAP, I knew the config surface cold, and it worked reliably. But it was also slow compared to what WireGuard can do, the config was heavier than it needed to be, and every time I had to add a peer it was a manual process I didn't enjoy. When I rebuilt the KDN Lab network on the Netgate 6100, I decided to move everything to WireGuard and do it properly from the start.

That meant not just swapping protocols but thinking through the full architecture: how peers connect, what they can reach based on which VLAN makes sense for them, how site-to-site tunnels connect the homelab to Vultr VPS nodes, and how all of it gets provisioned without doing it by hand every time. This post covers the whole journey.

## The Network Foundation

Before WireGuard made sense I needed the network segmented properly. The KDN Lab runs pfSense Plus on a Netgate 6100 with a full VLAN topology. The relevant segments for WireGuard are:

**LAN (VLAN 0)** is the native untagged network where core infrastructure lives. The Netgate 6100 itself, network switches, the Synology NAS, the UniFi UNAS Pro, a Lenovo ThinkCentre M70q Tiny as the central management workstation, and a Raspberry Pi 4 with an SSD in a USB enclosure all live here. This is the hardware layer everything else depends on and it stays isolated from the rest by design.

**MAIN (VLAN 10)** is the primary LAN. Trusted devices, homelab services, daily driver machines.

**Lab (VLAN 70)** is the infrastructure segment. Proxmox nodes, internal services, the monitoring stack. The stuff that doesn't need to be on the same segment as personal devices.

**IoT (VLAN 30)** is fully isolated. Smart home devices, cameras, nothing that should talk to anything it doesn't need to.

WireGuard peers land in their own dedicated tunnel subnet, `10.6.0.0/24`. That subnet doesn't map to any physical VLAN. It's the VPN layer and traffic from it into the rest of the network is controlled entirely by firewall rules on the assigned WireGuard interface. That separation is what makes split-tunnel and full-tunnel templates meaningful rather than cosmetic.

## Setting Up WireGuard on pfSense

WireGuard is a built-in package in pfSense Plus. Navigate to VPN, WireGuard, Tunnels, and add a new tunnel. The settings that matter at creation time are the listen port and the interface address.

I use UDP port 51820, the WireGuard default. The tunnel interface address is `10.6.0.1/24`, which makes pfSense the gateway for all peers on the VPN subnet. Click Generate to create the key pair. Copy that public key somewhere accessible because every peer config you create needs it.

After saving the tunnel, assign it as a proper interface. Go to Interfaces, Assignments, add the WireGuard tunnel, and give it a meaningful name. I call mine `WG_VPN`. Enable the interface and set the IPv4 address to `10.6.0.1/24`.

The WAN firewall rule comes next. Without it, inbound WireGuard packets get dropped before they reach the tunnel:

```
Interface:        WAN
Action:           Pass
Protocol:         UDP
Destination:      WAN address
Destination port: 51820
Description:      Allow WireGuard inbound
```

That's the entirety of what needs to be open on WAN. Everything else stays closed.

## Peer Templates: Split-Tunnel vs Full-Tunnel

This is where most WireGuard guides skip something important. The difference between split-tunnel and full-tunnel is not a pfSense setting. It's a single line in the client config file: the `AllowedIPs` value. That one field determines whether the client routes all its traffic through the VPN or only traffic destined for specific networks.

I have two templates I use depending on who the peer is and what they need.

**Split-tunnel** is for family members who need to reach internal services but shouldn't have all their traffic funneled through my home internet connection. The client only routes homelab-specific traffic through the VPN. Everything else, web browsing, streaming, general internet, goes out the client's own connection as normal.

```ini
[Interface]
PrivateKey = <client private key>
Address = 10.6.0.x/24
DNS = 10.6.0.1

[Peer]
PublicKey = <pfSense WireGuard public key>
Endpoint = your.public.ip:51820
AllowedIPs = 10.6.0.0/24, 192.168.1.0/24, 192.168.70.0/24
PersistentKeepalive = 25
```

The `AllowedIPs` list includes the WireGuard subnet itself and the specific VLANs the peer is allowed to reach. Anything not in that list routes outside the tunnel entirely.

**Full-tunnel** routes all traffic through the VPN. The client's default gateway becomes pfSense. This is what I use when I'm on an untrusted network and want everything encrypted, or when a device needs to appear to originate from the homelab for all purposes.

```ini
[Interface]
PrivateKey = <client private key>
Address = 10.6.0.x/24
DNS = 10.6.0.1

[Peer]
PublicKey = <pfSense WireGuard public key>
Endpoint = your.public.ip:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

The `0.0.0.0/0` captures all IPv4 traffic. The `::/0` captures all IPv6. With both present, the VPN becomes the client's entire internet path.

`PersistentKeepalive = 25` is in both templates and it matters for mobile devices. WireGuard is intentionally silent when there's no traffic, which means NAT mappings on cellular connections will expire after a period of inactivity. Sending a keepalive packet every 25 seconds keeps the mapping alive so the connection stays up when the screen goes idle.

## Firewall Rules on the WireGuard Interface

The WAN rule lets packets in. The rules on the WireGuard interface control what those peers can actually do once connected.

For split-tunnel peers who should reach MAIN and Lab but nothing else:

```
Interface:   WG_VPN
Action:      Pass
Protocol:    Any
Source:      WireGuard net
Destination: 192.168.1.0/24
Description: WireGuard peers to MAIN VLAN

Interface:   WG_VPN
Action:      Pass
Protocol:    Any
Source:      WireGuard net
Destination: 192.168.70.0/24
Description: WireGuard peers to Lab VLAN
```

For full-tunnel peers who need general internet access routed through pfSense, allow them to reach any destination:

```
Interface:   WG_VPN
Action:      Pass
Protocol:    Any
Source:      WireGuard net
Destination: Any
Description: WireGuard full-tunnel peers outbound
```

The order matters. pfSense evaluates rules top to bottom and stops at the first match. More specific rules go above more permissive ones.

IoT is intentionally absent from all WireGuard firewall rules. VPN peers have no path to the IoT VLAN by design. If something on IoT needs to be reachable remotely it goes through a different mechanism entirely.

## Adding Peers in pfSense

Each device gets a peer entry in pfSense under VPN, WireGuard, Peers. The required fields are the client's public key and the allowed IPs. In pfSense's peer config, allowed IPs means the IP addresses pfSense will accept traffic from for that peer. For a remote access client this is just the client's assigned tunnel IP with a `/32` mask:

```
Peer public key:  <client public key>
Allowed IPs:      10.6.0.x/32
Description:      jdoe-macbook
```

Assigning sequential IPs manually works fine at small scale. The Ansible playbook handles this automatically at larger scale, which I'll get to.

## Distributing Configs as QR Codes

For mobile devices, QR codes are the right way to deliver WireGuard configs. The WireGuard app on iOS and Android can scan a QR code and import the full config without the user touching a keyboard. Install `qrencode` and generate one from any config file:

```bash
qrencode -t ansiutf8 < peer-jdoe-phone.conf
```

This prints the QR code directly in the terminal. Point a phone camera at it, import in the WireGuard app, done. For family members who are not going to edit a text file, this is the difference between the VPN actually getting used and not.

## Site-to-Site: Extending to Vultr

Remote access peers connect individual devices. Site-to-site tunnels connect entire networks. I have two Vultr VPS nodes, both running Ubuntu, that need to reach homelab services and have homelab infrastructure reach them in return. Herald runs Twenty CRM for my wife's business. The other node handles outbound routing for specific traffic. Both need to be on the same private network as the homelab without any public exposure of the services they're running.

WireGuard handles this identically to remote access peers from pfSense's perspective. The difference is in the routing intent and the peer config on the VPS side.

### On the Vultr VPS (Ubuntu)

WireGuard is in the Ubuntu repos:

```bash
apt install wireguard
```

Generate a key pair on the VPS:

```bash
wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key
chmod 600 /etc/wireguard/private.key
```

Create the interface config at `/etc/wireguard/wg0.conf`. For site-to-site links I use `/30` subnets in a separate range from the client tunnel subnet. Point-to-point links only need two usable addresses and the smaller subnet makes it visually obvious this is an infrastructure link rather than a client peer:

```ini
[Interface]
PrivateKey = <VPS private key>
Address = 10.7.0.2/30
ListenPort = 51820
MTU = 1420

[Peer]
PublicKey = <pfSense public key>
Endpoint = your.pfsense.public.ip:51820
AllowedIPs = 192.168.1.0/24, 192.168.70.0/24, 10.7.0.0/30
PersistentKeepalive = 25
```

The `AllowedIPs` here lists the homelab networks this VPS should be able to reach through the tunnel. Adjust to match whatever subnets are relevant for that node's function.

Enable and start the interface:

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

### On pfSense for the Site-to-Site Peer

Back in pfSense, add a peer entry for the VPS with its public key. The allowed IPs should include the VPS tunnel address and any subnets on the VPS side that homelab traffic needs to reach. For nodes that are purely outbound clients of the homelab, just the tunnel IP is sufficient.

For routing to work in both directions, add a static route in pfSense under System, Routing, Static Routes. Point the route for the VPS tunnel subnet at the WireGuard interface gateway. pfSense needs this to know that traffic destined for `10.7.0.2` should go through the WireGuard tunnel rather than out the default WAN gateway.

You may need to add the gateway manually under System, Routing, Gateways first, pointing at the VPS tunnel IP, before the static route option becomes available.

### Verifying the Tunnel

On the VPS, check live tunnel state:

```bash
wg show
```

A healthy output looks like this:

```
interface: wg0
  public key: <VPS public key>
  private key: (hidden)
  listening port: 51820

peer: <pfSense public key>
  endpoint: your.pfsense.ip:51820
  allowed ips: 192.168.1.0/24, 192.168.70.0/24, 10.7.0.0/30
  latest handshake: 8 seconds ago
  transfer: 1.23 GiB received, 456 MiB sent
```

A handshake timestamp within the last few minutes means the tunnel is live. If latest handshake is blank or stale, check that the public keys match on both sides, verify the endpoint address and port are correct, and confirm that UDP 51820 is open in the Vultr firewall group assigned to the instance, not just the host-level firewall.

From pfSense, ping the VPS tunnel IP:

```bash
ping 10.7.0.2
```

From the VPS, ping a homelab host:

```bash
ping 192.168.1.1
```

If both directions work, routing is correct. One direction working and the other not usually means a static route is missing or the allowed IPs on one side doesn't include the right subnets.

### Aliases That Save Sanity

Once a tunnel is up and you're living inside it day to day, the `wg-quick` and `systemctl` commands get repetitive fast. I keep a set of aliases in my dotfiles that land on every peer node via Ansible. They live in a clearly labeled block so they're easy to find and they cover everything I actually reach for:

```bash
# wireguard vpn
alias wgshow='sudo wg show'
alias wgup='sudo wg-quick up wg0'
alias wgdown='sudo wg-quick down wg0'
alias wgrestart='sudo wg-quick down wg0 && sudo wg-quick up wg0'
alias wgenable='sudo systemctl enable --now wg-quick@wg0'
alias wgdisable='sudo systemctl disable --now wg-quick@wg0'
alias wgstatus='sudo systemctl status wg-quick@wg0'
alias wgpinghome='ping 10.6.0.1'
alias wgpinglan='ping 192.168.1.1'
alias wgmyip='curl -4 ifconfig.me'
alias wgmyip6='curl -6 ifconfig.me'
```

`wgshow` is the one I use most. A quick glance at the handshake timestamp and transfer stats tells you immediately whether the tunnel is healthy without reading through `systemctl status` output. `wgrestart` is the fast path when something needs a kick without thinking about the systemd invocation. The ping aliases are sanity checks: `wgpinghome` hits the WireGuard gateway on pfSense, `wgpinglan` hits a host on the main LAN. If both respond you're through the tunnel and routing is correct. `wgmyip` and `wgmyip6` confirm which exit IP the node is using, which matters when you're debugging whether traffic is routing through the tunnel or leaking out the local interface.

These get deployed automatically by the dotfiles role in Ansible so every new VPS node or peer device has them from the first login. That's the kind of small quality-of-life detail that makes managing a fleet of WireGuard nodes not feel like work.

## MTU

WireGuard adds overhead to each packet which reduces the effective MTU inside the tunnel. Leave this at default and you'll eventually see large transfers behave strangely, especially with services that don't handle path MTU discovery well.

The standard starting point is 1420 on both ends. It's already in the VPS config above. On pfSense it's set in the interface assignment configuration for the WireGuard interface. If you're running over a connection with an already-reduced MTU, like PPPoE or certain cloud provider uplinks, you may need to go lower. Test with large pings to find the ceiling:

```bash
ping -M do -s 1400 192.168.1.1
```

If that returns fragmentation errors but smaller packet sizes succeed, reduce the MTU until it works cleanly.

## Ansible: Automating Peer Provisioning

Doing this manually for three peers is fine. Doing it for ten is tedious and error-prone. I have an Ansible role in my `ops_automation` repo on my private Gitea instance that handles the full provisioning workflow.

For each new peer the role generates a WireGuard key pair using `wg genkey` and `wg pubkey`, assigns the next available tunnel IP from the peer subnet, and renders the client config from a Jinja2 template applying either the split-tunnel or full-tunnel `AllowedIPs` depending on a variable set per peer. It then generates a QR code from the rendered config using `qrencode`, registers the peer on pfSense via the pfSense API with the public key and allowed IPs, and stores the rendered config and public key in a structured directory so there's a record of everything issued.

For the VPS nodes there's a separate role that handles the Ubuntu side: installing WireGuard from apt, generating keys, rendering `wg0.conf` from a template, enabling the `wg-quick@wg0` systemd service, and opening the WireGuard port in `ufw`. A fresh Vultr instance goes from bare Ubuntu to a live tunnel in under two minutes.

Adding a new peer or provisioning a new VPS node is a single Ansible command. It's idempotent too, so running it again on an existing peer changes nothing. Run it on a new one and everything gets created correctly the first time.

## The Current State

WireGuard on pfSense Plus as the hub. Split-tunnel configs for family devices that need homelab access without routing all their traffic through here. Full-tunnel available for when I'm on an untrusted network. Site-to-site tunnels keeping Herald and the other Vultr node on the same private network as everything else in the lab. Peer provisioning fully automated. The only manual step left is physically scanning a QR code.

It replaced OpenVPN completely and I haven't missed it once.
