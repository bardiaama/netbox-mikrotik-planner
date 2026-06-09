# MikroTik RouterOS Config Templates from NetBox

This reference shows how to translate NetBox data into RouterOS commands.
Always confirm physical interface names with the user before applying.

---

## 1. System identity from NetBox device name

```routeros
/system identity set name="hq-core-01"
# netbox device: hq-core-01 (id: 12)
```

---

## 2. IP addresses from NetBox IPAM

Pull from NetBox:
```python
ifaces = nb.dcim.interfaces.filter(device_id=device.id, enabled=True)
for iface in ifaces:
    ips = nb.ipam.ip_addresses.filter(interface_id=iface.id)
    for ip in ips:
        print(f"/ip address add address={ip.address} interface={iface.name} comment=\"nb:{ip.id}\"")
```

RouterOS output:
```routeros
# === IP Addresses ===
/ip address
add address=203.0.113.2/30 interface=ether1 comment="WAN - nb:101"
add address=10.10.10.1/24  interface=bridge-users comment="Users GW - nb:102"
add address=10.10.20.1/24  interface=bridge-servers comment="Servers GW - nb:103"
add address=10.0.0.1/32    interface=lo comment="Loopback - nb:104"
```

---

## 3. VLANs from NetBox IPAM

Pull VLANs and their prefixes:
```python
vlans = nb.ipam.vlans.filter(site_id=site.id, status="active")
for vlan in vlans:
    prefix = next(nb.ipam.prefixes.filter(vlan_id=vlan.id, status="active"), None)
    gw = f"{prefix.prefix.split('/')[0].rsplit('.', 1)[0]}.1/{prefix.prefix.split('/')[1]}" if prefix else ""
    print(f"""
/interface vlan add vlan-id={vlan.vid} name=vlan{vlan.vid} interface=bridge comment="{vlan.name}"
/ip address add address={gw} interface=vlan{vlan.vid} comment="{vlan.name} GW - nb:{vlan.id}"
""")
```

RouterOS output:
```routeros
# === VLANs ===
/interface bridge
add name=bridge-core fast-forward=no comment="Core bridge"

/interface bridge port
add bridge=bridge-core interface=ether2 pvid=1
add bridge=bridge-core interface=ether3 pvid=1

/interface vlan
add name=vlan10 vlan-id=10 interface=bridge-core comment="USERS - nb:10"
add name=vlan20 vlan-id=20 interface=bridge-core comment="SERVERS - nb:20"
add name=vlan30 vlan-id=30 interface=bridge-core comment="MGMT - nb:30"
add name=vlan99 vlan-id=99 interface=bridge-core comment="NATIVE/TRUNK - nb:99"

# === VLAN IP addresses ===
/ip address
add address=10.10.10.1/24 interface=vlan10 comment="USERS GW"
add address=10.10.20.1/24 interface=vlan20 comment="SERVERS GW"
add address=10.10.30.1/24 interface=vlan30 comment="MGMT GW"
```

Bridge VLAN table (802.1Q) for switch chips (hEX, CRS, CSS):
```routeros
/interface bridge vlan
add bridge=bridge-core tagged=bridge-core,ether2 vlan-ids=10,20,30,99
add bridge=bridge-core tagged=bridge-core untagged=ether3 vlan-ids=10
```

---

## 4. DHCP pools from NetBox prefixes

NetBox prefixes with `is_pool=True` map to DHCP server configs.

```python
pools = nb.ipam.prefixes.filter(site_id=site.id, is_pool=True, status="active")
for p in pools:
    import ipaddress
    net = ipaddress.ip_network(p.prefix, strict=False)
    hosts = list(net.hosts())
    pool_start = str(hosts[10])   # skip first 10 for static
    pool_end   = str(hosts[-2])   # skip last for broadcast buffer
    gateway    = str(hosts[0])    # .1
    dns        = "8.8.8.8,2.1.1.1"
    vlan_name  = f"vlan{p.vlan.vid}" if p.vlan else "unknown"
    print(f"""
/ip pool add name=pool-{vlan_name} ranges={pool_start}-{pool_end}
/ip dhcp-server add name=dhcp-{vlan_name} interface={vlan_name} address-pool=pool-{vlan_name} lease-time=12h disabled=no
/ip dhcp-server network add address={p.prefix} gateway={gateway} dns-server={dns} comment="nb:{p.id}"
""")
```

RouterOS output:
```routeros
# === DHCP Server ===
/ip pool
add name=pool-vlan10 ranges=10.10.10.11-10.10.10.254
add name=pool-vlan20 ranges=10.10.20.11-10.10.20.254

/ip dhcp-server
add name=dhcp-vlan10 interface=vlan10 address-pool=pool-vlan10 lease-time=12h disabled=no
add name=dhcp-vlan20 interface=vlan20 address-pool=pool-vlan20 lease-time=6h  disabled=no

/ip dhcp-server network
add address=10.10.10.0/24 gateway=10.10.10.1 dns-server=10.10.30.53,8.8.8.8 comment="USERS - nb:201"
add address=10.10.20.0/24 gateway=10.10.20.1 dns-server=10.10.30.53,8.8.8.8 comment="SERVERS - nb:202"
```

---

## 5. Static routes from NetBox prefixes

NetBox prefixes with role "Static Route" or a custom field `next_hop`:

```routeros
# === Static Routes ===
/ip route
add dst-address=0.0.0.0/0      gateway=203.0.113.1 comment="Default via ISP - nb:301"
add dst-address=10.20.0.0/16   gateway=10.10.30.254 comment="Remote site via MPLS - nb:302"
add dst-address=192.168.100.0/24 gateway=10.10.30.1 type=blackhole comment="Null route - nb:303"
```

For ISP with BGP, document the upstream ASN in NetBox circuit/provider and use:
```routeros
/routing bgp connection
add name=ISP-PEER as=65001 remote.address=203.0.113.1/32 remote.as=65000 \
    local.role=ebgp output.network=bgp-networks comment="Irancell - circuit nb:401"
```

---

## 6. WireGuard from NetBox

Store WireGuard data in NetBox custom fields on device or interface:
- Device custom field: `wg_public_key`, `wg_listen_port`
- Peer custom field: `wg_peer_pubkey`, `wg_endpoint`, `wg_allowed_ips`

Or use NetBox tags: tag WireGuard interfaces as `wireguard` for easy filtering.

```routeros
# === WireGuard ===
/interface wireguard
add name=wg0 listen-port=51820 private-key="<PRIVATE_KEY>" comment="WG tunnel - nb:iface:501"

/interface wireguard peers
add interface=wg0 \
    public-key="<PEER_PUBKEY>" \
    endpoint-address=peer.example.com \
    endpoint-port=51820 \
    allowed-address=10.100.0.2/32,10.20.0.0/16 \
    persistent-keepalive=25 \
    comment="branch01 - nb:device:45"

/ip address
add address=10.100.0.1/30 interface=wg0 comment="WG tunnel address - nb:ip:601"
```

---

## 7. Firewall rules from network segments

Use NetBox VLAN roles and prefix descriptions to generate zone-based rules:

```routeros
# === Firewall Filter Rules ===
# Pattern: users can reach internet and servers; servers cannot initiate to users
/ip firewall filter

# Allow established/related
add chain=forward connection-state=established,related action=accept comment="Allow established"
add chain=forward connection-state=invalid action=drop comment="Drop invalid"

# USERS (vlan10) -> Internet: allow
add chain=forward in-interface=vlan10 out-interface=ether1 action=accept comment="USERS->WAN"

# USERS (vlan10) -> SERVERS (vlan20): allow HTTP/HTTPS/DNS
add chain=forward in-interface=vlan10 out-interface=vlan20 \
    protocol=tcp dst-port=80,443,53 action=accept comment="USERS->SERVERS http/https"
add chain=forward in-interface=vlan10 out-interface=vlan20 \
    protocol=udp dst-port=53 action=accept comment="USERS->SERVERS DNS"

# SERVERS (vlan20) -> USERS (vlan10): deny
add chain=forward in-interface=vlan20 out-interface=vlan10 action=drop comment="SERVERS->USERS blocked"

# MGMT (vlan30): allow all to all (admin VLAN)
add chain=forward in-interface=vlan30 action=accept comment="MGMT unrestricted"

# Default: drop forward
add chain=forward action=drop comment="Default drop"

# NAT masquerade for internet
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade comment="NAT to WAN"
```

---

## 8. ISP / PPPoE concentrator pattern

For ISPs managing customer PPPoE sessions:

```routeros
# NetBox: prefix 100.64.0.0/10 as CG-NAT pool, customer prefixes as /32 or /29

# === PPPoE Server ===
/ip pool
add name=pppoe-cgnat ranges=100.64.1.0-100.64.254.254

/interface pppoe-server server
add service-name=internet interface=ether2 \
    address-pool=pppoe-cgnat \
    authentication=chap,mschap2 \
    max-sessions=500 \
    keepalive-timeout=60 \
    disabled=no

# CG-NAT
/ip firewall nat
add chain=srcnat src-address=100.64.0.0/10 out-interface=ether1 \
    action=masquerade comment="CE-NAT for PPPoE customers"

# Per-customer static IP from NetBox (for business customers)
/ppp secret
add name=customer-biz01 password=secret service=pppoe \
    local-address=100.64.0.1 remote-address=203.0.113.10 \
    comment="Biz customer - nb:device:78"
```

---

## 9. CHR-specific setup

```routeros
# License check
/system license print

# Cloud-specific: get public IP automatically
/ip cloud set ddns-enabled=yes update-time=yes

# Typical CHR interface mapping (confirm with provider):
# ether1 = WAN/public
# ether2 = internal (if multi-NIC)

# Disable unused services
/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes   # disable if using winbox/ssh only
set api disabled=yes   # enable only if using API

# SSH hardening
/ip ssh set strong-crypto=yes forwarding-enabled=no
```

---

## 10. Full script generator pattern

When generating a complete config, use this structure:

```python
def generate_mikrotik_config(device_name, nb):
    device = nb.dcim.devices.get(name=device_name)
    lines = [
        f"# Config generated from NetBox for: {device.name}",
        f"# Device ID: {device.id} | Site: {device.site.name}",
        f"# Generated: {datetime.now().isoformat()}",
        "",
        "# === System ===",
        f'/system identity set name="{device.name}"',
        "",
    ]

    # IP Addresses
    lines.append("# === Interfaces & IPs ===")
    for iface in nb.dcim.interfaces.filter(device_id=device.id, enabled=True):
        for ip in nb.ipam.ip_addresses.filter(interface_id=iface.id):
            lines.append(
                f'/ip address add address={ip.address} interface={iface.name} '
                f'comment="{iface.description or iface.name} nb:{ip.id}"'
            )

    # VLANs
    # ... (add vlan loop here)

    # Routes
    lines.append("\n# === Routes ===")
    # ... (add route loop here)

    return "\n".join(lines)
```

---

## Quick reference: interface types

| Physical | RouterOS name | NetBox type |
|---|---|---|
| Copper GbE | ether1..etherN | 1000base-t |
| SFP+ 10G | sfp-sfpplus1.. | 10gbase-x-sfpp |
| SFP28 25G | sfp28-1.. | 25gbase-x-sfp28 |
| QSFP+ 40G | qsfpplus1-1.. | 40gbase-x-qsfpp |
| WiFi 2.4G | wlan1 | ieee802.11a |
| WiFi 5G | wlan2 | ieee802.11ac |
| Loopback | lo | virtual |
| Bridge | bridge | bridge |
| Bond/LAG | bond1 | lag |
| VLAN | vlanN | virtual |
| WireGuard | wgN | virtual |
