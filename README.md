# NetBox ↔ MikroTik Network Planner

A Claude Cowork skill for planning networks, documenting them in **NetBox** via API, and generating **MikroTik RouterOS** configurations from that data.

## What it does

- **Phase 1 — Plan**: Helps design IP addressing, VLAN layout, routing topology, and VPN overlays
- **Phase 2 — Document**: Creates NetBox objects via REST API (sites, devices, interfaces, prefixes, VLANs, circuits) in the correct dependency order
- **Phase 3 — Generate**: Pulls data from NetBox and produces ready-to-paste RouterOS commands

## Supported environments

| Environment | Details |
|---|---|
| CHR / Cloud | MikroTik CHR on Proxmox, VMware, cloud providers |
| ISP / Carrier | PPPoE, CG-NAT, BGP peering, VRF per customer |
| Enterprise / Datacenter | Rack elevation, OOB management, device roles |
| Home lab | Single-site, tags-based, Proxmox/Docker |

## Installation

1. Download `netbox-mikrotik-planner.skill`
2. In Claude Cowork, go to **Settings → Capabilities → Skills**
3. Drag and drop the `.skill` file to install

## Usage

Once installed, the skill triggers automatically when you mention:
- `NetBox`, `IPAM`, `DCIM`, `network documentation`
- `MikroTik config`, `RouterOS`, `generate config`
- `IP planning`, `VLAN design`, `prefix allocation`
- `plan my network`, `document my infrastructure`

### Example prompts

```
I have a NetBox instance at 192.168.1.10:8000. 
Help me document my 3-site network with 4 VLANs per site 
and generate MikroTik configs for the core routers.
```

```
Generate RouterOS firewall rules from my NetBox VLAN segments.
My users are on VLAN 10, servers on VLAN 20, management on VLAN 30.
```

```
I'm setting up a small ISP with PPPoE. 
Help me plan the address space in NetBox and generate the MikroTik concentrator config.
```

## Structure

```
netbox-mikrotik-planner/
├── SKILL.md                          # Main skill instructions
└── references/
    ├── netbox-api.md                 # NetBox REST API reference (Python + curl)
    └── mikrotik-templates.md         # RouterOS config templates from NetBox data
```

## Requirements

- NetBox instance with API access (token required)
- MikroTik RouterOS device (any model or CHR)
- Python `pynetbox` or `requests` library for automation scripts

## Author

Built with [Claude Cowork](https://claude.ai) — [@bardiaama](https://github.com/bardiaama)
