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
- vlan-configuration
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
- pfsense-api
- pfsense-restapi
- pfrest
- rest-api
- api-automation
- unifi
- unifi-cloudkey
- cloudkey-gen2-plus
- third-party-gateway
- 2-5gbps
- network-segmentation
- kids-vlan
- guest-vlan
- iot-vlan
- lab-vlan
---

I ran OpenVPN for years. I had it wired into LDAP, I knew the config surface cold, and it worked reliably. But it was also slow compared to what WireGuard can do, the config was heavier than it needed to be, and every time I had to add a peer it was a manual process I didn't enjoy. When I rebuilt the KDN Lab network on the Netgate 6100, I decided to move everything to WireGuard and do it properly from the start.

That meant not just swapping protocols but thinking through the full architecture: how peers connect, what they can reach based on which VLAN makes sense for them, how site-to-site tunnels connect the homelab to Vultr VPS nodes, and how all of it gets provisioned without doing it by hand every time. This post covers the whole journey.

## The Network Foundation

Before WireGuard made sense I needed the network segmented properly. The KDN Lab runs pfSense Plus on a Netgate 6100 with a full VLAN topology. The Netgate 6100 has four 2.5Gbps ports and one 10Gbps SFP+ port, which is one of the reasons I chose it. Most home routers ship with a single 1Gbps WAN port and that's the end of the conversation. The 6100 gives you the option to reassign ports freely in pfSense, so I moved the WAN off the default port and onto one of the 2.5Gbps interfaces to take full advantage of the connection coming in. The interface assignment page in pfSense is where that remapping happens: WAN, LAN, LAN2, LAN3 and OPT4 are all physical ports on the 6100, and you can swap which physical port handles which logical role without touching anything else in the config.

The wireless and VLAN side runs through a UniFi CloudKey Gen2 Plus, rack-mounted as a 1U unit. The CloudKey manages the UniFi APs and switches but since pfSense is the gateway rather than a UniFi Dream Machine, the CloudKey is operating in third-party gateway mode. That has one practical implication worth knowing about: VLANs have to be configured and enabled on both sides. You define the VLAN in pfSense, create the corresponding network in the UniFi controller, and tag it on the appropriate switch ports and SSIDs. If you only do it in pfSense the VLAN exists at the routing layer but nothing is actually tagging traffic into it. If you only do it in UniFi the controller thinks the network exists but pfSense doesn't know about it. Both sides have to match.

Here is a simplified view of how the full topology fits together:

```
                           INTERNET
                               |
                      +------------------+
                      |   Netgate 6100   |
                      |  pfSense Plus    |
                      |  2.5G ports      |
                      +--------+---------+
                               |
              +----------------+----------------+
              |                                 |
    +---------+----------+           +----------+---------+
    |   WireGuard Hub    |           |    VLANs / LAN     |
    |   wg0: 10.6.0.1    |           |                    |
    +--------------------+           | Main  v10          |
              |                      | Kids  v20          |
              |                      | IoT   v30          |
              |                      | Guest v50          |
              |                      | RemoteCorp v15     |
              |                      | Lab   v70          |
              |                      +----------+---------+
              |                                 |
              |                      +----------+---------+
              |                      | UniFi CloudKey G2+ |
              |                      | APs / Switches     |
              |                      | (tagged VLANs)     |
              |                      +--------------------+
              |
    +---------+------------------------------------------+
    |         WireGuard Peers  (10.6.0.0/24)             |
    |                                                    |
    |  [Split-tunnel]            [Full-tunnel]           |
    |  MacBook / Phone           Laptop (untrusted net)  |
    |  10.6.0.x/24              10.6.0.x/24              |
    |  reaches: Main, Lab        reaches: everything     |
    |                                                    |
    |  [VPS peer - same subnet]                          |
    |  Nexus   10.6.0.22/24                              |
    |  reaches: Main, Kids, Lab, Herald subnet           |
    +----------------------------------------------------+

    [VPS peer - dedicated subnet]
    +----------------------------------------------------+
    |  Herald  10.7.0.2/24                               |
    |  Vultr / Ubuntu                                    |
    |  reaches: Main, Kids, Lab, Nexus subnet            |
    +----------------------------------------------------+

    pfSense <======> Nexus   (encrypted UDP :51820)
    pfSense <======> Herald  (encrypted UDP :51820)
```

The `<=====>` links are WireGuard tunnels, all encrypted UDP over port 51820. These two nodes illustrate two valid approaches to how you assign VPS peers. Nexus lives on the same `10.6.0.0/24` tunnel subnet as client peers, just at a higher IP. Herald lives on its own dedicated `10.7.0.0/24` subnet, keeping its traffic cleanly separated from the client peer range. Both connect to pfSense as standard peers. There is no special site-to-site mode in WireGuard. The protocol is peer-to-peer by design and the routing intent is handled entirely by `AllowedIPs` on each side.

The current VLAN layout across the lab:

**LAN (VLAN 0)** is the native untagged network where core infrastructure lives. The Netgate 6100 itself, network switches, the Synology NAS, the UniFi UNAS Pro, a Lenovo ThinkCentre M70q Tiny as the central management workstation, and a Raspberry Pi 4 with an SSD in a USB enclosure all live here. This is the hardware layer everything else depends on and it stays isolated from the rest by design.

**Main (VLAN 10)** is the primary LAN. Trusted devices, homelab services, daily driver machines.

**Kids (VLAN 20)** is a dedicated segment for kids devices with its own firewall policy.

**IoT (VLAN 30)** is fully isolated. Smart home devices, cameras, nothing that should talk to anything it doesn't need to.

**Guest (VLAN 50)** is internet-only access for visitors. No path to internal networks.

**RemoteCorp (VLAN 15)** handles corporate VPN traffic, keeping work devices separated from personal infrastructure.

**Lab (VLAN 70)** is the infrastructure segment. Proxmox nodes, internal services, the monitoring stack. The stuff that doesn't need to be on the same segment as personal devices.

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

Create the interface config at `/etc/wireguard/wg0.conf`. Each VPS node connects as a standard peer just like any other WireGuard client. The difference is in what `AllowedIPs` contains and which subnet the node's tunnel address sits on.

Nexus sits on the same `10.6.0.0/24` tunnel subnet as regular client peers, just at a higher IP to make its role obvious at a glance:

```ini
[Interface]
PrivateKey = <Nexus private key>
Address = 10.6.0.22/24
ListenPort = 51820
MTU = 1420

[Peer]
PublicKey = <pfSense public key>
PresharedKey = <preshared key>
Endpoint = pfsense.yourdomain.cloud:51820
AllowedIPs = 192.168.1.0/24, 192.168.20.0/24, 192.168.70.0/24, 10.7.0.0/24
PersistentKeepalive = 25
```

Herald runs on its own dedicated `10.7.0.0/24` subnet. Putting it on a separate range keeps Herald's traffic cleanly separated from the client peer range and makes firewall rules and routing easier to reason about. It also uses DNS fallback to Cloudflare and Google alongside the internal resolver so it can resolve both internal and external names regardless of tunnel state:

```ini
[Interface]
PrivateKey = <Herald private key>
Address = 10.7.0.2/24
DNS = 192.168.1.1, 1.1.1.1, 8.8.8.8
MTU = 1420

[Peer]
PublicKey = <pfSense public key>
PresharedKey = <preshared key>
Endpoint = pfsense.yourdomain.cloud:51820
AllowedIPs = 10.7.0.0/24, 192.168.10.0/24, 192.168.20.0/24, 192.168.70.0/24
PersistentKeepalive = 25
```

Both configs include a `PresharedKey` on top of the public key exchange. This is an optional but worthwhile extra layer that adds a symmetric key into the handshake, providing post-quantum resistance against future attacks on the asymmetric cryptography. Generate one with `wg genpsk` and add the same value to both the peer config on the VPS and the peer entry on pfSense.

The `AllowedIPs` on each node lists the homelab subnets it needs to reach. Anything not in that list routes out the VPS's own internet connection rather than through the tunnel.

Enable and start the interface:

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

### On pfSense for the Site-to-Site Peer

Back in pfSense, add a peer entry for the VPS with its public key. The allowed IPs should include the VPS tunnel address and any subnets on the VPS side that homelab traffic needs to reach. For nodes that are purely outbound clients of the homelab, just the tunnel IP is sufficient.

For routing to work in both directions, add static routes in pfSense under System, Routing, Static Routes. For Nexus, pfSense already knows about `10.6.0.22` since it lives in the same `10.6.0.0/24` tunnel subnet. For Herald on `10.7.0.0/24`, you need a static route pointing that subnet at the WireGuard interface gateway so pfSense knows to send traffic for that range through the tunnel rather than out the default WAN gateway.

You may need to add the gateway manually under System, Routing, Gateways first, pointing at the peer tunnel IP, before the static route option becomes available.

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
  allowed ips: 192.168.1.0/24, 192.168.20.0/24, 192.168.70.0/24, 10.7.0.0/24
  latest handshake: 8 seconds ago
  transfer: 1.23 GiB received, 456 MiB sent
```

A handshake timestamp within the last few minutes means the tunnel is live. If latest handshake is blank or stale, check that the public keys match on both sides, verify the endpoint address and port are correct, and confirm that UDP 51820 is open in the Vultr firewall group assigned to the instance, not just the host-level firewall.

From pfSense, ping the Nexus tunnel IP:

```bash
ping 10.6.0.22
```

Or the Herald tunnel IP:

```bash
ping 10.7.0.2
```

From either VPS, ping a homelab host to confirm the return path works:

```bash
ping 192.168.1.1
```

If both directions work, routing is correct. One direction working and the other not usually means a static route is missing or the `AllowedIPs` on one side doesn't include the right subnets.

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

Doing this manually for three peers is fine. Doing it for ten is tedious and error-prone. The full role lives in my private `ops_automation` repo on Gitea, but everything you need to replicate it is here.

One prerequisite worth calling out first: the Ansible role talks to pfSense via its REST API, which means you need [pfSense-pkg-RESTAPI](https://pfrest.org/) installed on your pfSense instance. It is not available in the pfSense package manager, including on pfSense Plus, so you install it manually from the shell. SSH into pfSense and run the one-liner from the [quickstart](https://github.com/pfrest/pfSense-pkg-RESTAPI#quickstart):

```bash
pkg add https://github.com/pfrest/pfSense-pkg-RESTAPI/releases/latest/download/pfSense-pkg-RESTAPI.pkg
```

Once installed the REST API is available at `https://your-pfsense-host/api/v2/` and includes full WireGuard peer management endpoints. The role uses that API to register new peers directly in pfSense without you touching the GUI.

**Mac prerequisites**

The role runs locally on your Mac. You need `wireguard-tools` for key generation and `qrencode` for QR code output:

```bash
brew install wireguard-tools qrencode
```

**Role structure**

```
roles/
  wireguard_peer/
    defaults/
      main.yml
    tasks/
      main.yml
    templates/
      client.conf.j2

playbooks/
  wireguard_peer.yml
  wireguard_deploy.yml

group_vars/
  all/
    all.yml       # wireguard: block lives here
    vault.yml     # wireguard_api_key goes here

peers/            # gitignored: generated configs land here
  alice-iphone/
    client.conf
    qr.png
```

**group_vars/all/all.yml**

The `wireguard:` block centralises every value the role needs. Fill in your own `pfsense_wg_pubkey` from VPN, WireGuard, Tunnels in the pfSense GUI, and your external DDNS hostname for `pfsense_endpoint`. The `allowed_ips` map is what drives the split vs full tunnel decision at render time:

```yaml
wireguard:
  pfsense_host: "https://pfsense.yourdomain.cloud"
  pfsense_tunnel: "tun_wg0"
  pfsense_wg_pubkey: "your-pfsense-public-key"
  pfsense_endpoint: "pfsense.yourdomain.cloud:51820"
  keepalive: 25
  generate_psk: true
  dns_servers: "192.168.1.1, 1.1.1.1, 8.8.8.8"
  vpn_subnet_base: "10.6.0"
  vpn_subnet_cidr: "10.6.0.0/24"
  vpn_ip_start: 2
  vpn_peer_mask: 24
  allowed_ips:
    split: "10.6.0.0/24, 192.168.1.0/24, 192.168.10.0/24, 192.168.20.0/24, 192.168.70.0/24"
    full: "0.0.0.0/0, ::/0"
```

The pfSense API key goes into vault, never in plaintext:

```bash
ansible-vault edit group_vars/all/vault.yml
```

```yaml
wireguard_api_key: "your-pfsense-api-key"
```

**defaults/main.yml**

The default tunnel mode is split. Override at runtime with `-e "tunnel=full"`:

```yaml
---
tunnel: "split"
peer_name: ""
```

**tasks/main.yml**

The tasks file is the full workflow in sequence: dependency checks, key generation, IP assignment via pfSense API, peer registration, config rendering, and QR code output. The IP assignment step is the part that makes this genuinely useful at scale. It queries pfSense for all existing peers, parses the used IP octets, and finds the next available one automatically so you never have to track IP assignments manually:

```yaml
# 1. Check dependencies
- name: Check wg is installed
  ansible.builtin.command: which wg
  register: wg_check
  changed_when: false
  failed_when: wg_check.rc != 0

- name: Check qrencode is installed
  ansible.builtin.command: which qrencode
  register: qr_check
  changed_when: false
  failed_when: qr_check.rc != 0

# 2. Generate keys
- name: Generate WireGuard private key
  ansible.builtin.command: wg genkey
  register: wg_privkey_result
  changed_when: true
  no_log: true

- name: Derive public key from private key
  ansible.builtin.shell: "echo '{{ wg_privkey_result.stdout }}' | wg pubkey"
  register: wg_pubkey_result
  changed_when: false
  no_log: true

- name: Generate pre-shared key
  ansible.builtin.command: wg genpsk
  register: wg_psk_result
  changed_when: true
  no_log: true
  when: wireguard.generate_psk | bool

- name: Set key facts
  ansible.builtin.set_fact:
    peer_privkey: "{{ wg_privkey_result.stdout }}"
    peer_pubkey: "{{ wg_pubkey_result.stdout }}"
    peer_psk: "{{ wg_psk_result.stdout | default('') }}"
  no_log: true

# 3. Determine next available IP
- name: Fetch existing peers from pfSense
  ansible.builtin.uri:
    url: "{{ wireguard.pfsense_host }}/api/v2/vpn/wireguard/peers"
    method: GET
    headers:
      x-api-key: "{{ wireguard_api_key }}"
      Content-Type: "application/json"
    validate_certs: true
    return_content: true
  register: existing_peers_response
  no_log: true

- name: Parse used IP last octets
  ansible.builtin.set_fact:
    used_octets: "{{ existing_peers_response.json.data
      | map(attribute='allowedips') | flatten
      | map(attribute='address')
      | select('search', wireguard.vpn_subnet_base)
      | map('regex_replace', '^.+\.', '')
      | map('int') | list }}"

- name: Find next available IP octet
  ansible.builtin.set_fact:
    next_octet: "{{ range(wireguard.vpn_ip_start, 255)
      | reject('in', used_octets) | first }}"

- name: Set peer IP
  ansible.builtin.set_fact:
    peer_ip: "{{ wireguard.vpn_subnet_base }}.{{ next_octet }}"

- name: Show assigned IP
  ansible.builtin.debug:
    msg: "Assigning {{ peer_ip }}/{{ wireguard.vpn_peer_mask }} to {{ peer_name }}"

# 4. Register peer in pfSense via REST API
- name: POST new peer to pfSense
  ansible.builtin.uri:
    url: "{{ wireguard.pfsense_host }}/api/v2/vpn/wireguard/peer"
    method: POST
    headers:
      x-api-key: "{{ wireguard_api_key }}"
      Content-Type: "application/json"
    body_format: json
    body:
      tun: "{{ wireguard.pfsense_tunnel }}"
      enabled: true
      descr: "{{ peer_name }}"
      publickey: "{{ peer_pubkey }}"
      presharedkey: "{{ peer_psk }}"
      persistentkeepalive: "{{ wireguard.keepalive }}"
      allowedips:
        - address: "{{ peer_ip }}"
          mask: "{{ wireguard.vpn_peer_mask }}"
    validate_certs: true
    status_code: 200
  register: pfsense_response
  no_log: true

- name: Confirm peer created
  ansible.builtin.debug:
    msg: "Peer {{ peer_name }} registered in pfSense (ID: {{ pfsense_response.json.data.id }})"

# 5. Create output directory
- name: Ensure peers directory exists
  ansible.builtin.file:
    path: "peers/{{ peer_name }}"
    state: directory
    mode: "0700"

# 6. Render client config
- name: Render client.conf
  ansible.builtin.template:
    src: client.conf.j2
    dest: "peers/{{ peer_name }}/client.conf"
    mode: "0600"

# 7. Generate QR code
- name: Generate QR code PNG
  ansible.builtin.command: >
    qrencode -t PNG -s 5
    -o peers/{{ peer_name }}/qr.png
    -r peers/{{ peer_name }}/client.conf
  changed_when: true

- name: Print QR code to terminal
  ansible.builtin.command: >
    qrencode -t ansiutf8
    -r peers/{{ peer_name }}/client.conf
  register: qr_terminal
  changed_when: false

- name: Display QR code
  ansible.builtin.debug:
    msg: "{{ qr_terminal.stdout_lines }}"

# 8. Summary
- name: Peer summary
  ansible.builtin.debug:
    msg:
      - "Peer:    {{ peer_name }}"
      - "IP:      {{ peer_ip }}/{{ wireguard.vpn_peer_mask }}"
      - "Tunnel:  {{ tunnel }}"
      - "Config:  peers/{{ peer_name }}/client.conf"
      - "QR PNG:  peers/{{ peer_name }}/qr.png"
```

**templates/client.conf.j2**

The template uses the `tunnel` variable to pull the correct `AllowedIPs` value straight from the `wireguard.allowed_ips` map in vars. The preshared key block only renders if one was generated:

```jinja2
# {{ peer_name }}
[Interface]
PrivateKey = {{ peer_privkey }}
Address = {{ peer_ip }}/{{ wireguard.vpn_peer_mask }}
DNS = {{ wireguard.dns_servers }}

[Peer]
PublicKey = {{ wireguard.pfsense_wg_pubkey }}
{% if peer_psk %}
PresharedKey = {{ peer_psk }}
{% endif %}
AllowedIPs = {{ wireguard.allowed_ips[tunnel] }}
Endpoint = {{ wireguard.pfsense_endpoint }}
PersistentKeepalive = {{ wireguard.keepalive }}
```

**Running it**

Add `peers/` to `.gitignore` first since generated configs contain private keys:

```bash
echo "peers/" >> .gitignore
```

Provision a split-tunnel peer (the default):

```bash
ansible-playbook playbooks/wireguard_peer.yml -e "peer_name=alice-iphone"
```

Provision a full-tunnel peer:

```bash
ansible-playbook playbooks/wireguard_peer.yml -e "peer_name=bob-laptop tunnel=full"
```

The role queries pfSense for existing peers, finds the next available IP automatically, registers the peer via the REST API, renders the config, and outputs both a QR code PNG and a terminal QR code you can scan immediately. The whole thing runs in seconds.

**Deploying WireGuard to a VPS**

For VPS nodes the workflow is two steps. First provision the peer config, then deploy WireGuard to the host using the generated config:

```bash
# Step 1: provision peer in pfSense and generate config locally
ansible-playbook playbooks/wireguard_peer.yml -e "peer_name=herald-crm"

# Step 2: install WireGuard on the VPS and drop the config in place
ansible-playbook playbooks/wireguard_deploy.yml -l herald -e "peer_name=herald-crm"
```

The deploy playbook installs WireGuard from apt, copies the generated `client.conf` to `/etc/wireguard/wg0.conf`, and enables `wg-quick@wg0` as a systemd service. Verify the tunnel is up after it runs:

```bash
ssh user@vps-ip sudo wg show
```

You should see the pfSense peer listed with a recent handshake timestamp.

**Debugging**

To list all peers currently registered in pfSense without creating a new one:

```bash
ansible-playbook playbooks/wireguard_peer.yml -e "peer_name=debug" --tags debug_peers
```

This is useful for spotting orphaned peers that are occupying IP addresses without a corresponding active device.

## The Current State

WireGuard on pfSense Plus as the hub. Split-tunnel configs for family devices that need homelab access without routing all their traffic through here. Full-tunnel available for when I'm on an untrusted network. Site-to-site tunnels keeping Herald and the other Vultr node on the same private network as everything else in the lab. Peer provisioning fully automated. The only manual step left is physically scanning a QR code.

It replaced OpenVPN completely and I haven't missed it once.
