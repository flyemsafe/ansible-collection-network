# rhdc.network.vlan_bridge

Configure VLAN interfaces and bridges on network hosts for VM/container networking.

## Description

This role configures VLAN subinterfaces and bridges on Linux hosts using NetworkManager (`nmcli`). It's designed for KVM hypervisors, container hosts, or any system requiring VLAN-tagged bridge networking.

**Key features:**
- Create VLAN subinterfaces on physical NICs (e.g., `eno1.10`, `eno1.11`)
- Create bridge interfaces with clean naming (`br-vlan10`, `br-vlan11`)
- Enslave VLAN interfaces to bridges
- Handle interface name changes (e.g., hardware migration)
- Idempotent (only changes when needed)
- Support multiple VLANs in single role invocation

## Requirements

- **OS**: RHEL 8+, Rocky Linux 8+, Fedora 38+
- **Packages**: NetworkManager (nmcli)
- **Network**: UniFi switch port must be configured as trunk (see rhdc.network.unifi role)

## Role Variables

### Required Variables

```yaml
rhdc_network_vlan_bridges:
  - vlan_id: 10
    parent_interface: eno1
    bridge_name: br-vlan10
    description: "Management VLAN"  # Optional

  - vlan_id: 11
    parent_interface: eno1
    bridge_name: br-vlan11
    description: "Lab network"
```

### Optional Variables

```yaml
# Auto-start connections on boot (default: true)
rhdc_network_vlan_bridge_autoconnect: true

# Bridge STP (Spanning Tree Protocol) setting (default: false)
rhdc_network_vlan_bridge_stp: false

# Remove existing connections before creating (default: false)
rhdc_network_vlan_bridge_force: false
```

## Dependencies

None.

## Example Playbook

### Basic Usage

```yaml
---
- name: Configure VLAN bridges on KVM hosts
  hosts: kvm_hosts
  become: true

  roles:
    - role: rhdc.network.vlan_bridge
      vars:
        rhdc_network_vlan_bridges:
          - vlan_id: 10
            parent_interface: eno1
            bridge_name: br-vlan10

          - vlan_id: 11
            parent_interface: eno1
            bridge_name: br-vlan11

          - vlan_id: 60
            parent_interface: eno1
            bridge_name: br-vlan60
```

### Complete Configuration for kvmzfs01

```yaml
---
- name: Fix kvmzfs01 VLAN bridges
  hosts: kvmzfs01.lab.rodhouse.net
  become: true

  roles:
    - role: rhdc.network.vlan_bridge
      vars:
        rhdc_network_vlan_bridges:
          - {vlan_id: 10, parent_interface: eno1, bridge_name: br-vlan10, description: "Management"}
          - {vlan_id: 11, parent_interface: eno1, bridge_name: br-vlan11, description: "Lab"}
          - {vlan_id: 12, parent_interface: eno1, bridge_name: br-vlan12, description: "Server"}
          - {vlan_id: 14, parent_interface: eno1, bridge_name: br-vlan14, description: "Database"}
          - {vlan_id: 20, parent_interface: eno1, bridge_name: br-vlan20, description: "Guest"}
          - {vlan_id: 21, parent_interface: eno1, bridge_name: br-vlan21, description: "IoT"}
          - {vlan_id: 22, parent_interface: eno1, bridge_name: br-vlan22, description: "Lab Test"}
          - {vlan_id: 23, parent_interface: eno1, bridge_name: br-vlan23, description: "Development"}
          - {vlan_id: 24, parent_interface: eno1, bridge_name: br-vlan24, description: "Staging"}
          - {vlan_id: 31, parent_interface: eno1, bridge_name: br-vlan31, description: "VPN"}
          - {vlan_id: 50, parent_interface: eno1, bridge_name: br-vlan50, description: "DMZ"}
          - {vlan_id: 60, parent_interface: eno1, bridge_name: br-vlan60, description: "Storage"}
```

### Force Recreate (Hardware Migration)

```yaml
---
- name: Recreate VLAN bridges after hardware migration
  hosts: kvm_hosts
  become: true

  roles:
    - role: rhdc.network.vlan_bridge
      vars:
        rhdc_network_vlan_bridge_force: true  # Remove old connections
        rhdc_network_vlan_bridges:
          - vlan_id: 11
            parent_interface: eno1  # New interface name
            bridge_name: br-vlan11
```

## How It Works

For each VLAN configuration:

1. **Create bridge interface:**
   ```bash
   nmcli connection add type bridge \
     con-name br-vlan11 \
     ifname br-vlan11 \
     stp no \
     autoconnect yes
   ```

2. **Create VLAN interface as bridge slave:**
   ```bash
   nmcli connection add type vlan \
     con-name vlan11 \
     dev eno1 \
     id 11 \
     master br-vlan11 \
     slave-type bridge \
     autoconnect yes
   ```

3. **Activate connections:**
   ```bash
   nmcli connection up br-vlan11
   nmcli connection up vlan11
   ```

## Validation

After running the role:

### Check VLAN interfaces
```bash
ip link show | grep eno1\.
# Should show: eno1.10, eno1.11, etc.
```

### Check bridges
```bash
ip link show | grep br-vlan
# Should show: br-vlan10, br-vlan11, etc.
```

### Check bridge slaves
```bash
bridge link show
# Should show VLAN interfaces as slaves of bridges
```

### Check NetworkManager connections
```bash
nmcli connection show | grep -E '(vlan|br-vlan)'
# Should show all VLAN and bridge connections
```

## Troubleshooting

### VLAN interface doesn't activate
**Cause:** Parent interface doesn't exist
**Fix:** Verify correct parent interface name:
```bash
ip link show | grep -E '^[0-9]+: eno'
```

### Bridge has no slaves
**Cause:** VLAN connection didn't activate
**Fix:** Check VLAN connection status:
```bash
nmcli connection show vlan11
# Look for "connection.master" and "connection.slave-type"
```

### Connection already exists error
**Solution:** Set `rhdc_network_vlan_bridge_force: true` to remove old connections

### VMs can't reach network
**Cause:** UniFi switch port not configured as trunk
**Fix:** Use `rhdc.network.unifi` role to configure switch port as trunk

## Related Roles

- **rhdc.network.unifi** - Configure UniFi switch ports (trunk/access)
- **rhdc.libvirt.network** - Attach libvirt networks to bridges

## Migration from Old Naming

If migrating from old bridge naming (`vlan10br0` → `br-vlan10`):

1. **Stop VMs using old bridges**
2. **Run role with new naming**
3. **Update libvirt network definitions** (use rhdc.libvirt.network role)
4. **Start VMs on new bridges**

## License

Apache-2.0

## Author

Rodrique Heron (<rheron@redhat.com>)
RHDC Infrastructure Team

## Links

- **Collection**: https://github.com/flyemsafe/rhdc-ansible/tree/main/ansible-collection-network
- **Issue**: https://github.com/flyemsafe/rodhouse-datacenter/issues/312
- **Related Issues**: #311 (UniFi trunk), #313 (libvirt network), #314 (kvmzfs01 fix)

---

*Part of the RHDC Ansible Collections*
