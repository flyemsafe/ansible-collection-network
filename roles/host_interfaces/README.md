# rhdc.network.host_interfaces

Configure Linux host network interfaces with RHDC-specific patterns and best practices.

## Description

This role provides a simplified, opinionated interface for configuring network interfaces on RHDC hosts. It wraps `fedora.linux_system_roles.network` to provide:

- **RHDC network presets**: Pre-configured settings for storage, management, and IPMI networks
- **Simplified syntax**: Define interfaces with purpose-based configuration
- **Consistent standards**: MTU 9000 for storage, isolated networks, etc.
- **Host-based configuration**: Define network config in host_vars, not in playbooks

## Requirements

- Fedora 38+ or RHEL/CentOS 8+
- NetworkManager (default on modern systems)
- Collection: `fedora.linux_system_roles` (installed automatically)

## Role Variables

### Required Variables

**`rhdc_network_interfaces`** - List of network interfaces to configure

Each interface definition supports:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `name` | Yes | - | Interface name (e.g., eno8, enp2s0) |
| `ip` | Yes* | - | IP address in CIDR notation |
| `purpose` | No | - | Preset: storage, management, ipmi |
| `type` | No | ethernet | Interface type |
| `state` | No | up | Interface state (up/down) |
| `mtu` | No | 1500 | MTU size (9000 for storage) |
| `autoconnect` | No | true | Auto-connect on boot |
| `gateway` | No | - | Gateway IP (auto-omitted for storage) |
| `dns` | No | - | List of DNS servers |
| `no_gateway` | No | false | Explicitly disable gateway |

*Required for static IP configuration

### Optional Variables

**`rhdc_network_provider`** - Network provider (default: `nm`)
- `nm` - NetworkManager (recommended)
- `initscripts` - Legacy network-scripts (RHEL 6/7)

**`rhdc_network_allow_restart`** - Allow NetworkManager restart (default: `false`)

**`rhdc_network_remove_undefined`** - Remove undefined interfaces (default: `false`)

## Network Presets

The role includes pre-configured presets for common RHDC networks:

### Storage Network (`purpose: storage`)
- Network: 172.23.13.0/24 (VLAN 13)
- MTU: 9000 (jumbo frames)
- No default gateway (isolated network)
- High-speed storage traffic

### Management Network (`purpose: management`)
- Network: 172.23.11.0/24 (VLAN 11)
- MTU: 1500
- Administrative access

### IPMI Network (`purpose: ipmi`)
- Network: 172.23.10.0/24 (VLAN 10)
- MTU: 1500
- Out-of-band management

## Dependencies

- `fedora.linux_system_roles.network` (declared in meta/main.yml)

## Examples

### Example 1: Single 10G Storage Interface

Define in host_vars:

```yaml
# inventory/host_vars/kvmzfs03.lab.rodhouse.net.yml
rhdc_network_interfaces:
  - name: eno8
    purpose: storage
    ip: 172.23.13.4/24
```

Playbook:

```yaml
- name: Configure network interfaces
  hosts: kvmzfs03.lab.rodhouse.net
  roles:
    - rhdc.network.host_interfaces
```

### Example 2: Multiple Interfaces

```yaml
# inventory/host_vars/tazama.lab.rodhouse.net.yml
rhdc_network_interfaces:
  - name: enp2s0
    purpose: storage
    ip: 172.23.13.20/24

  - name: eno1
    purpose: management
    ip: 172.23.11.145/24
    gateway: 172.23.11.1
    dns:
      - 172.23.11.1
      - 1.1.1.1
```

### Example 3: Custom MTU Without Preset

```yaml
rhdc_network_interfaces:
  - name: eno8
    ip: 172.23.13.4/24
    mtu: 9000
    no_gateway: true
```

### Example 4: Group-Based Configuration

Configure all storage servers at once:

```yaml
# inventory/group_vars/storage_servers.yml
rhdc_network_interfaces:
  - name: "{{ storage_interface }}"
    purpose: storage
    ip: "{{ storage_ip }}"

# inventory/host_vars/kvmzfs03.lab.rodhouse.net.yml
storage_interface: eno8
storage_ip: 172.23.13.4/24

# inventory/host_vars/kvmzfs04.lab.rodhouse.net.yml
storage_interface: eno8
storage_ip: 172.23.13.5/24
```

## Comparison: Before and After

### Before (Snowflake Playbook)

```yaml
# playbooks/configure-kvmzfs03-10g-storage.yml
- name: Configure 10G storage network on kvmzfs03
  hosts: kvmzfs03.lab.rodhouse.net
  tasks:
    - name: Check if eno8 exists
      ansible.builtin.command: ip link show eno8
      # ... 50 more lines of nmcli commands
```

### After (Reusable Role)

```yaml
# inventory/host_vars/kvmzfs03.lab.rodhouse.net.yml
rhdc_network_interfaces:
  - name: eno8
    purpose: storage
    ip: 172.23.13.4/24

# playbooks/configure-storage-network.yml
- name: Configure storage network
  hosts: storage_servers
  roles:
    - rhdc.network.host_interfaces
```

## Architecture

```
rhdc.network.host_interfaces (This Role)
    |
    ├─> Validates RHDC interface definitions
    ├─> Applies purpose-based presets (storage, management, ipmi)
    ├─> Transforms to network_connections format
    |
    └─> fedora.linux_system_roles.network (Upstream Role)
         |
         └─> NetworkManager (nm provider)
              └─> nmcli / D-Bus API
```

## Testing

Test the role configuration without applying:

```bash
ansible-playbook playbooks/configure-storage-network.yml --check --diff
```

Verify after applying:

```bash
# Check NetworkManager connections
nmcli connection show

# Check interface configuration
ip addr show eno8

# Test connectivity
ping -c 3 172.23.13.1
```

## Troubleshooting

### Interface not created

Check that the physical interface exists:
```bash
ip link show
```

### Configuration not applied

Check NetworkManager status:
```bash
systemctl status NetworkManager
nmcli connection show
```

### MTU not applied

Verify MTU on interface:
```bash
ip link show eno8 | grep mtu
```

## License

MIT

## Author

Rodhouse Datacenter Infrastructure Team
