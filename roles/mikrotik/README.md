# Ansible Role: rhdc.network.mikrotik

Configure MikroTik RouterOS switches with 10G SFP+ trunk, VLAN support, and storage network configuration.

## Description

This role automates the configuration of MikroTik CRS-series switches running RouterOS. It supports:

- **10G SFP+ trunk port** to upstream switch (UniFi)
- **VLAN configuration** (Management VLAN 11, Storage VLAN 13)
- **Bridge and VLAN filtering** for proper traffic isolation
- **Storage port configuration** for server 10G connectivity
- **System setup** (identity, NTP, timezone, jumbo frames)
- **Monitoring** (optional SNMP, configuration backup)

## Requirements

### RouterOS (Not SwOS)
- **CRITICAL**: This role requires RouterOS, **NOT** SwOS
- SwOS does not have SSH access or CLI capabilities
- To switch from SwOS to RouterOS:
  1. Download RouterOS from [mikrotik.com/download](https://mikrotik.com/download)
  2. Upload via SwOS web interface: System → RouterOS
  3. Reboot to RouterOS

### Ansible Collections
```yaml
collections:
  - community.routeros
  - ansible.netcommon
```

Install with:
```bash
ansible-galaxy collection install community.routeros ansible.netcommon
```

### Credentials
Store MikroTik password in ansible-vault:
```bash
ansible-vault edit inventory/group_vars/all/vault
```

Add:
```yaml
vault_mikrotik_password: "YourSecurePassword"
```

## Role Variables

### Connection Settings
```yaml
mikrotik_hostname: "172.23.11.3"
mikrotik_username: "admin"
mikrotik_password: "{{ vault_mikrotik_password }}"
```

### Management VLAN
```yaml
mikrotik_mgmt_vlan_id: 11
mikrotik_mgmt_ip: "172.23.11.3/24"
mikrotik_mgmt_gateway: "172.23.11.1"
```

### Storage VLAN
```yaml
mikrotik_storage_vlan_id: 13
mikrotik_storage_vlan_name: "vlan13-storage"
```

### Trunk Configuration
```yaml
mikrotik_trunk_enabled: true
mikrotik_trunk_interface: "sfp-sfpplus8"  # 10G SFP+ port
mikrotik_trunk_vlans:
  - 11  # Management
  - 13  # Storage
```

### Storage Ports
```yaml
mikrotik_storage_ports:
  - name: "sfp-sfpplus1"
    description: "kvmzfs01-storage"
  - name: "sfp-sfpplus2"
    description: "kvmzfs02-storage"
  # ... etc
```

See `defaults/main.yml` for full list of variables.

## Dependencies

None.

## Example Playbook

### Basic Usage
```yaml
---
- name: Configure MikroTik CRS309
  hosts: localhost
  gather_facts: false

  roles:
    - role: rhdc.network.mikrotik
```

### Custom Configuration
```yaml
---
- name: Configure MikroTik with custom settings
  hosts: localhost
  gather_facts: false

  roles:
    - role: rhdc.network.mikrotik
      vars:
        mikrotik_hostname: "172.23.11.3"
        mikrotik_identity: "storage-switch-01"
        mikrotik_snmp_enabled: true
        mikrotik_trunk_interface: "sfp-sfpplus8"
```

### Tags
Run specific configuration tasks:
```bash
# Only configure bridges and VLANs
ansible-playbook playbook.yml --tags mikrotik_vlan

# Only configure storage ports
ansible-playbook playbook.yml --tags mikrotik_storage

# Skip monitoring configuration
ansible-playbook playbook.yml --skip-tags mikrotik_monitoring
```

## Architecture

### Network Topology
```
UniFi USW-Pro-24 Port 25 (10G SFP+, trunk)
    |
    | VLANs 11, 13 (tagged)
    |
MikroTik CRS309 (sfp-sfpplus8)
    |
    ├── VLAN 11 (Management) → 172.23.11.3/24
    └── VLAN 13 (Storage)
            |
            ├── sfp-sfpplus1 → kvmzfs01
            ├── sfp-sfpplus2 → kvmzfs02
            ├── sfp-sfpplus3 → kvmzfs03
            ├── sfp-sfpplus4 → kvmzfs07
            └── sfp-sfpplus5 → akili
```

### VLAN Configuration
- **VLAN 11 (VL11_MGMT)**: Management traffic, switch access
- **VLAN 13 (VL13_STORAGE)**: 10G storage network, server-to-server

### Bridges
- **bridge-trunk**: Trunk port with VLAN filtering enabled
- **bridge-storage**: Storage ports (VLAN 13 only)

## Verification

After running the playbook, verify configuration:

```bash
# SSH to MikroTik
ssh admin@172.23.11.3

# Check interfaces
/interface print

# Check bridges
/interface bridge print

# Check VLANs
/interface vlan print

# Check IP addresses
/ip address print

# Check bridge ports
/interface bridge port print

# Check VLAN filtering
/interface bridge vlan print

# Monitor port statistics
/interface monitor-traffic sfp-sfpplus8
```

## Troubleshooting

### Cannot connect via SSH
- **Verify RouterOS is running** (not SwOS)
  - SwOS has no SSH access
  - Check web interface: http://172.23.11.3
  - Look for "SwOS" or "RouterOS" in interface

### Configuration fails
- **Check credentials** in vault
- **Verify network connectivity**:
  ```bash
  ping 172.23.11.3
  ```
- **Check community.routeros collection** is installed:
  ```bash
  ansible-galaxy collection list | grep routeros
  ```

### Port not working after configuration
- **Check physical connection** (cable, SFP module)
- **Verify trunk on UniFi side**:
  - Port 25 must be configured as trunk
  - VLANs 11 and 13 must be allowed
  - STP must be enabled
- **Check MikroTik logs**:
  ```bash
  /log print where topics~"error"
  ```

## Migration from SwOS

If your MikroTik is currently running SwOS:

1. **Access web interface**: http://172.23.11.3
2. **Download RouterOS**: Get from [mikrotik.com/download](https://mikrotik.com/download)
3. **Upload RouterOS**: System → RouterOS → Upload file
4. **Reboot**: Switch will boot to RouterOS
5. **Initial config**: Set admin password via web interface
6. **Run playbook**: Use this role to complete configuration

**Note**: Migration from SwOS to RouterOS preserves physical port connections but resets all configuration. You will need to reconfigure from scratch.

## Security Considerations

- Store passwords in ansible-vault (never plaintext)
- Use strong admin password
- Restrict management access to VLAN 11 only
- Disable unused services
- Enable firewall if exposed to untrusted networks

## License

MIT

## Author

RHDC Infrastructure Team
