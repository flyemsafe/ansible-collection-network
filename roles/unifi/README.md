# rhdc.network.unifi

Ansible role for managing UniFi Network infrastructure via the UniFi Controller API.

## Description

This role provides automation for UniFi network infrastructure management, including:
- Switch port VLAN configuration (access ports - single VLAN)
- Trunk port configuration (multiple tagged VLANs)
- Network/VLAN management
- Infrastructure querying and discovery
- Wireless network configuration

## Requirements

- Ansible 2.15 or higher
- Access to a UniFi Controller (version 7.0+)
- Valid UniFi Controller credentials stored in Ansible Vault

## Role Variables

### Required Variables

Stored in Ansible Vault at `inventory/group_vars/all/vault`:

```yaml
network:
  unifi:
    controller: "172.23.11.5"  # IP address only, no https:// or port
    username: "admin"
    password: "your-password"
```

The role automatically constructs the full URL: `https://{{ network.unifi.controller }}:8443`

### Controller Configuration

```yaml
unifi_controller_url: "https://172.23.11.5:8443"  # UniFi Controller URL
unifi_site: "default"                              # UniFi site name
unifi_validate_certs: false                        # SSL certificate validation
```

### Action Selection

```yaml
unifi_action: "query"  # Options: query, switch_port, trunk_port, network
```

### Switch Port Configuration

When `unifi_action: "switch_port"`:

```yaml
unifi_switch_device_id: "abc123def456"     # UniFi device ID
unifi_switch_port_idx: 2                    # Port number (1-based)
unifi_switch_port_network_id: "xyz789"     # Network configuration ID
unifi_switch_port_name: "Server Port"      # Optional: Port label
unifi_switch_port_profile_id: "profile1"   # Optional: Port profile ID
unifi_switch_port_security: false          # Optional: Port security
```

### Trunk Port Configuration (Multiple VLANs)

When `unifi_action: "trunk_port"`:

```yaml
unifi_switch_device_id: "abc123def456"           # UniFi device ID
unifi_switch_port_idx: 24                         # Port number (1-based)
unifi_trunk_port_native_network_id: "xyz789"     # Native/untagged VLAN network ID
unifi_trunk_port_tagged_network_ids:             # Tagged VLAN network IDs
  - "abc123"
  - "def456"
  - "ghi789"
unifi_switch_port_name: "Trunk to kvmzfs01"      # Optional: Port label
unifi_switch_port_profile_id: "profile1"         # Optional: Port profile ID
unifi_switch_port_security: false                # Optional: Port security
```

## Dependencies

None

## Example Playbooks

### Query UniFi Infrastructure

```yaml
---
- name: Query UniFi Infrastructure
  hosts: localhost
  gather_facts: false

  roles:
    - role: rhdc.network.unifi
      vars:
        unifi_action: "query"
        unifi_controller_username: "{{ vault_unifi_username }}"
        unifi_controller_password: "{{ vault_unifi_password }}"
```

### Change Switch Port VLAN

```yaml
---
- name: Move tazama to VL12_SERVER
  hosts: localhost
  gather_facts: false

  tasks:
    # First, query to get network IDs
    - name: Query UniFi for network information
      ansible.builtin.include_role:
        name: rhdc.network.unifi
      vars:
        unifi_action: "query"

    # Get VL12_SERVER network ID
    - name: Find VL12_SERVER network ID
      ansible.builtin.set_fact:
        vl12_network_id: "{{ (unifi_query_results.networks | selectattr('name', 'equalto', 'VL12_SERVER') | first)._id }}"

    # Get switch device ID
    - name: Find USW-Pro-24 device ID
      ansible.builtin.set_fact:
        switch_device_id: "{{ (unifi_query_results.switches | selectattr('name', 'equalto', 'USW-Pro-24') | first)._id }}"

    # Update port configuration
    - name: Change port 2 to VLAN 12
      ansible.builtin.include_role:
        name: rhdc.network.unifi
      vars:
        unifi_action: "switch_port"
        unifi_switch_device_id: "{{ switch_device_id }}"
        unifi_switch_port_idx: 2
        unifi_switch_port_network_id: "{{ vl12_network_id }}"
        unifi_switch_port_name: "tazama - VL12_SERVER"
```

### Configure Trunk Port (Multiple VLANs)

```yaml
---
- name: Configure trunk port for kvmzfs01
  hosts: localhost
  gather_facts: false

  tasks:
    # Query UniFi for network IDs
    - name: Query UniFi infrastructure
      ansible.builtin.include_role:
        name: rhdc.network.unifi
      vars:
        unifi_action: "query"

    # Get network IDs for VLANs 10-60
    - name: Build list of tagged VLAN network IDs
      ansible.builtin.set_fact:
        tagged_vlan_ids: "{{ unifi_query_results.networks
          | selectattr('name', 'in', ['VL10_MGMT', 'VL11_LAB', 'VL12_SERVER', 'VL20_STORAGE', 'VL30_DMZ', 'VL40_IOT', 'VL50_GUEST', 'VL60_ISOLATED'])
          | map(attribute='_id') | list }}"

    # Get native VLAN network ID (VLAN 10)
    - name: Find VL10_MGMT network ID for native VLAN
      ansible.builtin.set_fact:
        native_vlan_id: "{{ (unifi_query_results.networks | selectattr('name', 'equalto', 'VL10_MGMT') | first)._id }}"

    # Get switch device ID
    - name: Find USW-Pro-24 device ID
      ansible.builtin.set_fact:
        switch_device_id: "{{ (unifi_query_results.switches | selectattr('name', 'equalto', 'USW-Pro-24') | first)._id }}"

    # Configure trunk port
    - name: Configure port 24 as trunk (VLANs 10-60)
      ansible.builtin.include_role:
        name: rhdc.network.unifi
      vars:
        unifi_action: "trunk_port"
        unifi_switch_device_id: "{{ switch_device_id }}"
        unifi_switch_port_idx: 24
        unifi_trunk_port_native_network_id: "{{ native_vlan_id }}"
        unifi_trunk_port_tagged_network_ids: "{{ tagged_vlan_ids }}"
        unifi_switch_port_name: "Trunk to kvmzfs01"
```

## Usage

### Installation

This role is part of the `rhdc.network` collection. Install the collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

Or install from source:

```bash
cd ~/Workspaces/datacenter/rhdc-ansible/ansible-collection-network
ansible-galaxy collection build
ansible-galaxy collection install rhdc-network-*.tar.gz
```

### Authentication

UniFi credentials are automatically loaded from vault. No need to pass them as variables.

To view/edit vault structure:
```bash
rhdc-vault-structure        # View structure only (safe)
rhdc-viewvault              # View with secrets (dangerous)
rhdc-editvault              # Edit vault
```

### Common Tasks

**Query all infrastructure:**
```bash
ansible-playbook playbooks/unifi-query.yml
```

**Change switch port VLAN:**
```bash
ansible-playbook playbooks/unifi-change-port-vlan.yml \
  -e "switch_name=USW-Pro-24" \
  -e "port_number=2" \
  -e "vlan_name=VL12_SERVER"
```

## API Endpoints Used

This role interacts with the following UniFi Controller API endpoints:

- `/api/login` - Authentication
- `/api/logout` - Session cleanup
- `/api/self/sites` - Site discovery
- `/api/s/{site}/stat/device` - Device information
- `/api/s/{site}/rest/networkconf` - Network/VLAN configuration
- `/api/s/{site}/rest/wlanconf` - Wireless networks
- `/api/s/{site}/rest/portconf` - Port profiles
- `/api/s/{site}/rest/device/{id}` - Device configuration updates

## Tags

- `unifi` - All tasks
- `unifi:auth` - Authentication only
- `unifi:query` - Infrastructure querying
- `unifi:switch_port` - Switch port configuration (access ports)
- `unifi:trunk_port` - Trunk port configuration (multiple VLANs)
- `unifi:network` - Network management
- `unifi:logout` - Session cleanup

## License

MIT

## Author

Rodhouse Datacenter Infrastructure Team

## See Also

- [UniFi Controller API Documentation](https://ubntwiki.com/products/software/unifi-controller/api)
- [rhdc.network Collection](https://github.com/flyemsafe/rhdc-ansible/tree/main/ansible-collection-network)
- [Rodhouse Datacenter Documentation](https://github.com/flyemsafe/rodhouse-datacenter)
