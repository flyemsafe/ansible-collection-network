# rhdc.network.bridge_vlan

Configure Linux bridges with VLAN sub-interfaces using NetworkManager for libvirt networking.

Supports **multiple trunk interfaces** - configure bridges on both lab and storage networks in a single playbook.

## Architecture

This role implements the RHDC "bridge per VLAN" architecture:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Physical Host (kvmzfs03, etc.)                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Lab Network (eno2) - Trunk port      Storage Network (enp2s0)     │
│       │                                      │                      │
│  ┌────┴────┐   ┌──────────┐            ┌────┴─────┐                │
│  │ rhdcbr0 │   │vlan10br0 │            │storagebr0│                │
│  │ (bridge)│   │ (bridge) │            │ (bridge) │                │
│  │         │   │          │            │          │                │
│  │ Has IP  │   │  No IP   │            │ Has IP   │                │
│  │172.23.11│   │ Layer 2  │            │172.23.13 │                │
│  │ .43/24  │   │  only    │            │ .43/24   │                │
│  │ +gateway│   │          │            │no gateway│                │
│  └────┬────┘   └────┬─────┘            └────┬─────┘                │
│       │             │                        │                      │
│  ┌────┴────┐   ┌────┴─────┐            ┌────┴─────┐                │
│  │  eno2   │   │ eno2.10  │            │ enp2s0   │                │
│  │ (slave) │   │ (VLAN)   │            │ (slave)  │                │
│  └─────────┘   └──────────┘            └──────────┘                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Key concepts:**
- **Main bridge (rhdcbr0)**: Has host IP, trunk interface enslaved directly
- **VLAN bridges (vlan10br0, etc.)**: Layer 2 only, no IP, VLAN sub-interface enslaved
- **VLAN sub-interfaces (eno2.10, etc.)**: Created with `ipv4.method disabled` (CRITICAL)
- **Storage bridge (storagebr0)**: Has IP but no gateway (local network only)

## Requirements

- RHEL 9 / Fedora 38+ with NetworkManager
- Physical NIC connected to switch trunk port (carrying tagged VLANs)
- Root/sudo access

## Role Variables

### Multiple Interfaces (Recommended)

Configure multiple trunk interfaces in a single playbook:

```yaml
rhdc_bridge_vlan_interfaces:
  # Lab network with VLANs
  - interface: "eno2"
    main_bridge:
      name: "rhdcbr0"
      ipv4_address: "172.23.11.43/24"
      ipv4_gateway: "172.23.11.1"
      ipv4_dns: ["172.23.11.1"]
      stp: false
    vlan_bridges:
      - { name: vlan10br0, vlan_id: 10 }
      - { name: vlan11br0, vlan_id: 11 }
      - { name: vlan12br0, vlan_id: 12 }

  # Storage network - simple bridge, no VLANs, no gateway
  - interface: "enp2s0"
    main_bridge:
      name: "storagebr0"
      ipv4_address: "172.23.13.43/24"
      ipv4_gateway: ""        # No gateway for local-only network
      ipv4_dns: []
      stp: false
    vlan_bridges: []          # No VLANs
```

### Legacy Single Interface (Backwards Compatible)

The original single-interface variables still work:

```yaml
# Physical interface connected to switch trunk port
rhdc_bridge_vlan_trunk_interface: "eno1"

# Main bridge configuration (native VLAN - carries host IP)
rhdc_bridge_vlan_main_bridge:
  name: "rhdcbr0"
  ipv4_address: "172.23.11.41/24"
  ipv4_gateway: "172.23.11.1"
  ipv4_dns: ["172.23.11.1"]  # Optional
  stp: false                  # Optional, default false

# VLAN bridges to create (Layer 2 only, no IP)
rhdc_bridge_vlan_bridges:
  - name: vlan10br0
    vlan_id: 10
```

### Common Options

```yaml
# Cleanup old/conflicting connections before configuring
rhdc_bridge_vlan_cleanup_connections:
  - "Wired connection 1"
  - "old-bridge"

# Skip validation checks (use with caution)
rhdc_bridge_vlan_skip_validation: false

# Bring up connections after creating them
rhdc_bridge_vlan_activate_connections: true

# Backup current network config before changes (creates restore script)
rhdc_bridge_vlan_create_backup: true          # default: true
rhdc_bridge_vlan_backup_dir: /root/network-backup  # default
```

## Backup and Recovery

This role automatically creates a backup of the current network configuration before making any changes. The backup includes:

- Current NetworkManager connections list
- IP configuration for the trunk interface
- Routing table
- DNS configuration
- **A restore script** that can revert networking to the original state

### Restore Script Location

```bash
# Latest restore script (symlink)
/root/network-backup/restore-network-latest.sh

# Timestamped backups
/root/network-backup/network-backup-<timestamp>-restore.sh
```

### How to Restore (via BMC/IPMI Console)

If something goes wrong and you lose network connectivity:

1. Access the host via BMC/IPMI console
2. Run the restore script:
   ```bash
   /root/network-backup/restore-network-latest.sh
   ```
3. The script will:
   - Remove all bridge and VLAN connections created by this role
   - Restore the original connection for the trunk interface
   - Bring up networking

### Disabling Backup

If you want to skip backup (not recommended):

```yaml
rhdc_bridge_vlan_create_backup: false
```

## Example Playbooks

### Multiple Interfaces: Lab + Storage Networks (Recommended)

```yaml
---
- name: Configure bridge networking for kvmzfs03
  hosts: kvmzfs03.lab.rodhouse.net
  become: true

  roles:
    - role: rhdc.network.bridge_vlan
      vars:
        rhdc_bridge_vlan_interfaces:
          # Lab network (primary) - with VLANs for VMs
          - interface: "eno2"
            main_bridge:
              name: "rhdcbr0"
              ipv4_address: "172.23.11.43/24"
              ipv4_gateway: "172.23.11.1"
              ipv4_dns: ["172.23.11.1"]
              stp: false
            vlan_bridges:
              - { name: vlan10br0, vlan_id: 10 }
              - { name: vlan11br0, vlan_id: 11 }
              - { name: vlan12br0, vlan_id: 12 }
              - { name: vlan14br0, vlan_id: 14 }

          # Storage network - simple bridge, no VLANs, no gateway
          - interface: "enp2s0"
            main_bridge:
              name: "storagebr0"
              ipv4_address: "172.23.13.43/24"
              ipv4_gateway: ""
              ipv4_dns: []
              stp: false
            vlan_bridges: []

        rhdc_bridge_vlan_cleanup_connections:
          - "Wired connection 1"
          - "Wired connection 2"
```

### Single Interface: Lab Network Only (Legacy)

```yaml
---
- name: Configure bridge networking for kvmzfs01
  hosts: kvmzfs01.lab.rodhouse.net
  become: true

  roles:
    - role: rhdc.network.bridge_vlan
      vars:
        rhdc_bridge_vlan_trunk_interface: "eno1"
        rhdc_bridge_vlan_main_bridge:
          name: "rhdcbr0"
          ipv4_address: "172.23.11.41/24"
          ipv4_gateway: "172.23.11.1"
          ipv4_dns: ["172.23.11.1"]
        rhdc_bridge_vlan_bridges:
          - name: vlan10br0
            vlan_id: 10
          - name: vlan11br0
            vlan_id: 11
          - name: vlan12br0
            vlan_id: 12
          - name: vlan14br0
            vlan_id: 14
        rhdc_bridge_vlan_cleanup_connections:
          - "Wired connection 1"
```

## Integration with libvirt

After running this role, create libvirt networks that reference the bridges:

```yaml
- name: Configure libvirt networks
  ansible.builtin.include_role:
    name: rhdc.libvirt.network
  vars:
    rhdc_libvirt_networks:
      - name: rhdcnet
        bridge: rhdcbr0
        autostart: true
      - name: vlan10
        bridge: vlan10br0
        autostart: true
      - name: vlan11
        bridge: vlan11br0
        autostart: true
```

## Verification Commands

After running the role (and rebooting), verify with:

```bash
# All bridges UP
ip link show type bridge | grep "state UP"

# Main bridge has IP
ip addr show rhdcbr0 | grep "inet "

# VLAN interfaces enslaved to bridges
bridge link show | grep "eno1\."

# No IPs on VLAN interfaces (should return nothing)
ip addr show | grep "eno1\." | grep "inet "

# NetworkManager connections
nmcli connection show
```

## Critical Details

### VLAN sub-interfaces MUST have `ipv4.method disabled`

Without this, NetworkManager defaults to DHCP. VLAN interfaces get IPs, bridge-slave activation fails, and bridges stay DOWN.

This role handles this automatically:
```bash
nmcli connection add type vlan ... ipv4.method disabled ipv6.method disabled
```

### VLAN sub-interfaces ARE bridge slaves

The VLAN connection specifies `connection.master` and `connection.slave-type`:
```bash
nmcli connection add type vlan ... connection.master vlan10br0 connection.slave-type bridge
```

Do NOT create separate bridge-slave connections for VLAN interfaces.

## Warning

**This role modifies network configuration.** If configuration is wrong, the host may lose network connectivity.

- Always have BMC/IPMI/console access available for recovery
- Test with `--check` first: `ansible-playbook playbook.yml --check --diff`
- A reboot is recommended after running this role

## Troubleshooting

### Bridge shows "state DOWN"

Check if slave interfaces are connected:
```bash
bridge link show master rhdcbr0
```

If empty, the trunk interface isn't enslaved properly.

### VLAN interface has IP address

The VLAN connection was created without `ipv4.method disabled`:
```bash
nmcli connection modify vlan10 ipv4.method disabled ipv6.method disabled
nmcli connection up vlan10
```

### "Connection activation failed"

Check NetworkManager logs:
```bash
journalctl -u NetworkManager -f
```

Common causes:
- Parent interface doesn't exist
- VLAN ID conflict
- Bridge doesn't exist yet

## License

MIT

## Author

RHDC Team
