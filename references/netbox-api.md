# NetBox REST API Reference

## Authentication

All requests require a token in the `Authorization` header:

```bash
curl -H "Authorization: Token $NETBOX_TOKEN" \
     -H "Content-Type: application/json" \
     "$NETBOX_URL/api/dcim/devices/"
```

Python with requests:
```python
import requests, os

NB_URL = os.environ["NETBOX_URL"].rstrip("/")
HEADERS = {
    "Authorization": f"Token {os.environ['NETBOX_TOKEN']}",
    "Content-Type": "application/json",
    "Accept": "application/json",
}

def nb_get(path, params=None):
    r = requests.get(f"{NB_URL}/api/{path}", headers=HEADERS, params=params)
    r.raise_for_status()
    return r.json()

def nb_post(path, data):
    r = requests.post(f"{NB_URL}/api/{path}", headers=HEADERS, json=data)
    r.raise_for_status()
    return r.json()

def nb_patch(path, obj_id, data):
    r = requests.patch(f"{NB_URL}/api/{path}{obj_id}/", headers=HEADERS, json=data)
    r.raise_for_status()
    return r.json()
```

Python with pynetbox (preferred):
```python
import pynetbox, os

nb = pynetbox.api(os.environ["NETBOX_URL"], token=os.environ["NETBOX_TOKEN"])
nb.http_session.verify = False  # if using self-signed cert
```

---

## Pagination

NetBox paginates at 50 results by default. Always handle pagination:

```python
def nb_get_all(path, params=None):
    params = params or {}
    params["limit"] = 200
    results = []
    url = f"{NB_URL}/api/{path}"
    while url:
        r = requests.get(url, headers=HEADERS, params=params)
        r.raise_for_status()
        data = r.json()
        results.extend(data["results"])
        url = data["next"]
        params = {}  # next URL already contains params
    return results
```

---

## DCIM (Data Center Infrastructure Management)

### Sites
```
GET  /api/dcim/sites/               list all sites
POST /api/dcim/sites/               create site
GET  /api/dcim/sites/?name=branch1  filter by name
```

Create a site:
```python
nb.dcim.sites.create({
    "name": "HQ",
    "slug": "hq",
    "status": "active",        # active | planned | staging | decommissioning | retired
    "description": "Headquarters - Tehran",
})
```

### Racks
```python
nb.dcim.racks.create({
    "site": site_id,
    "location": location_id,   # optional
    "name": "Rack-A1",
    "u_height": 42,
    "status": "active",
})
```

### Device Types (manufacturer + model)
```python
# First get or create manufacturer
mfr = nb.dcim.manufacturers.get(name="MikroTik")
if not mfr:
    mfr = nb.dcim.manufacturers.create({"name": "MikroTik", "slug": "mikrotik"})

# Create device type
dt = nb.dcim.device_types.create({
    "manufacturer": mfr.id,
    "model": "CCR2004-1G-12S+2XS",
    "slug": "ccr2004-1g-12s-2xs",
    "u_height": 1,
    "is_full_depth": True,
})
```

### Devices
```python
role = nb.dcim.device_roles.get(name="Router")
site = nb.dcim.sites.get(name="HQ")
dtype = nb.dcim.device_types.get(model="CCR2004-1G-12S+2XS")

device = nb.dcim.devices.create({
    "name": "hq-core-01",
    "device_type": dtype.id,
    "role": role.id,
    "site": site.id,
    "rack": rack_id,           # optional
    "position": 40,            # U position in rack (optional)
    "face": "front",
    "status": "active",
    "platform": platform_id,   # optional, e.g., "RouterOS"
    "primary_ip4": None,       # set after creating IPs
})
```

### Interfaces
```python
iface = nb.dcim.interfaces.create({
    "device": device.id,
    "name": "ether1",
    "type": "1000base-t",      # 1000base-t | 10gbase-x-sfpp | virtual | lag | bridge
    "enabled": True,
    "description": "WAN uplink to ISP",
    "mode": "access",          # access | tagged | tagged-all
    "untagged_vlan": vlan_id,  # for access mode
    # "tagged_vlans": [v1, v2] for tagged mode
})
```

Common interface types: `virtual`, `1000base-t`, `10gbase-x-sfpp`, `25gbase-x-sfp28`, `bridge`, `lag`

---

## IPAM (IP Address Management)

### VRFs
```python
vrf = nb.ipam.vrfs.create({
    "name": "CUSTOMER-A",
    "rd": "65001:100",         # route distinguisher
    "description": "Customer A VRF",
})
```

### Prefixes
```python
# Aggregate (top-level supernet)
nb.ipam.aggregates.create({
    "prefix": "10.0.0.0/8",
    "rir": rir_id,             # RIR object (RFC1918, IANA, etc.)
    "description": "Private address space",
})

# Site prefix
prefix = nb.ipam.prefixes.create({
    "prefix": "10.10.0.0/16",
    "site": site.id,
    "vrf": vrf_id,             # optional
    "status": "container",     # container | active | reserved | deprecated
    "description": "HQ address space",
    "is_pool": False,
})

# VLAN prefix
vlan_prefix = nb.ipam.prefixes.create({
    "prefix": "10.10.10.0/24",
    "site": site.id,
    "vlan": vlan_id,
    "status": "active",
    "is_pool": True,           # mark as pool for DHCP use
    "description": "Users VLAN 10",
})
```

### Get next available prefix
```python
parent = nb.ipam.prefixes.get(prefix="10.10.0.0/16")
available = nb.ipam.prefixes.available_prefixes.list(parent.id)
# Returns list of available child prefixes
```

### IP Addresses
```python
ip = nb.ipam.ip_addresses.create({
    "address": "10.10.10.1/24",
    "vrf": vrf_id,             # optional
    "status": "active",        # active | reserved | deprecated | dhcp | slaac
    "role": "loopback",        # loopback | secondary | anycast | vip | vrrp | hsrp | glbp | carp
    "assigned_object_type": "dcim.interface",
    "assigned_object_id": iface.id,
    "dns_name": "gw-users.hq.example.com",
    "description": "Users gateway",
})

# Set as primary IP on device
nb.dcim.devices.update([{"id": device.id, "primary_ip4": ip.id}])
```

### VLANs
```python
vg = nb.ipam.vlan_groups.create({
    "name": "HQ-VLANs",
    "slug": "hq-vlans",
    "site": site.id,
    "min_vid": 1,
    "max_vid": 4094,
})

vlan = nb.ipam.vlans.create({
    "vid": 10,
    "name": "USERS",
    "site": site.id,
    "group": vg.id,
    "status": "active",
    "role": role_id,           # optional: e.g., "User Access"
    "description": "End-user access VLAN",
})
```

---

## Circuits (for ISP/uplinks)

```python
# Create provider
provider = nb.circuits.providers.create({
    "name": "Irancell",
    "slug": "irancell",
    "asn": 44244,
    "account": "CUST-12345",
})

# Create circuit type
ct = nb.circuits.circuit_types.create({"name": "Internet", "slug": "internet"})

# Create circuit
circuit = nb.circuits.circuits.create({
    "cid": "IRC-001",          # circuit ID from provider
    "provider": provider.id,
    "type": ct.id,
    "status": "active",
    "commit_rate": 1000000,    # kbps — 1 Gbps
    "description": "Main internet uplink",
})
```

---

## Filtering tips

Most fields are filterable via query params:

```python
# All active devices at a site
nb.dcim.devices.filter(site="hq", status="active")

# IPs in a specific prefix
nb.ipam.ip_addresses.filter(parent="10.10.10.0/24")

# Interfaces on a device
nb.dcim.interfaces.filter(device_id=device.id)

# VLANs in a site
nb.ipam.vlans.filter(site_id=site.id)

# Tagged/custom field filtering
nb.dcim.devices.filter(tag="mikrotik")
```

---

## Custom fields

Add custom fields in NetBox UI: Customization → Custom Fields

Read/write them like regular fields:
```python
# Read
device = nb.dcim.devices.get(name="hq-core-01")
bgp_asn = device.custom_fields.get("bgp_asn")

# Write
nb.dcim.devices.update([{
    "id": device.id,
    "custom_fields": {"bgp_asn": 65001, "wireguard_port": 51820}
}])
```

---

## Useful lookup patterns

```python
# Get or create (safe upsert pattern)
def get_or_create(resource, lookup_field, lookup_value, create_data):
    existing = resource.get(**{lookup_field: lookup_value})
    if existing:
        return existing, False
    return resource.create(create_data), True

site, created = get_or_create(nb.dcim.sites, "slug", "hq", {
    "name": "HQ", "slug": "hq", "status": "active"
})

# Get all IPs assigned to a device
def get_device_ips(device_id):
    ifaces = nb.dcim.interfaces.filter(device_id=device_id)
    ips = []
    for iface in ifaces:
        ips.extend(nb.ipam.ip_addresses.filter(interface_id=iface.id))
    return ips

# Get all prefixes for a site
def get_site_prefixes(site_slug):
    return list(nb.ipam.prefixes.filter(site=site_slug, status="active"))
```
