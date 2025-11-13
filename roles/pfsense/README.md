# Ansible Role: pfSense

Manage pfSense firewall configuration via REST API and PHP shell.

## Description

This role provides a unified interface for managing pfSense configuration. It automatically prefers the REST API when available and falls back to PHP shell for operations not yet supported by the API.

## Requirements

- pfSense 2.7+ or pfSense Plus 24.03+
- pfSense REST API package installed (optional but recommended)
- SSH access to pfSense with admin privileges
- Ansible 2.14+

## Role Variables

### Connection Configuration

```yaml
# pfSense API connection (required)
pfsense_api_url: "https://{{ ansible_host }}:10443/api/v2"
pfsense_api_key: "{{ network.pfsense.api_key }}"  # Store in Ansible Vault
pfsense_hostname: "{{ inventory_hostname }}"
pfsense_validate_certs: false

# API vs PHP preference
pfsense_prefer_api: true  # Prefer API when available
```

### Feature Flags

```yaml
pfsense_manage_dhcp: false      # Enable DHCP management
pfsense_manage_dns: false       # Enable DNS management
pfsense_manage_firewall: false  # Enable firewall management
pfsense_manage_haproxy: false   # Enable HAProxy management
pfsense_manage_openvpn: false   # Enable OpenVPN management
pfsense_manage_backup: false    # Enable backup management
```

### OpenVPN Configuration

```yaml
# OpenVPN certificate management
pfsense_manage_openvpn_certificates: true  # Update certificates when enabled
pfsense_openvpn_server_name: "LOCAL VPN TUNNEL"
pfsense_openvpn_certificate_name: "rodhouse-dot-net-wildcard"
pfsense_openvpn_expiration_threshold: 30  # Days before expiration to update
```

### Backup Configuration

```yaml
pfsense_backup_dir: "/tmp/pfsense-backups"
pfsense_backup_prefix: "{{ pfsense_hostname }}"
pfsense_backup_retention_days: 30
```

### General Settings

```yaml
pfsense_auto_confirm: false  # Set to true to skip confirmation prompts
pfsense_verbose: false       # Enable verbose output for debugging
```

### DHCP Configuration

#### PXE Boot

```yaml
pfsense_dhcp_pxe_config:
  interface: "lan"
  next_server: "192.168.1.5"
  use_http_boot: true
  http_boot_url: "http://192.168.1.5/boot.ipxe"
  filename_bios: "boot.ipxe"
  filename_uefi: "http://192.168.1.5/boot.ipxe"
```

#### Static Mappings

```yaml
pfsense_dhcp_static_mappings:
  - interface: "lan"
    mac: "00:11:22:33:44:55"
    ipaddr: "192.168.1.100"
    hostname: "server01"
    domain: "lab.local"
    descr: "Production Server"
```

### Firewall Configuration

#### Aliases

```yaml
pfsense_firewall_aliases:
  - name: "INTERNAL_NETWORKS"
    type: "network"
    addresses:
      - "172.23.10.0/24"
      - "172.23.11.0/24"
      - "172.23.12.0/24"
    descr: "All internal networks"

  - name: "ADMIN_PORTS"
    type: "port"
    addresses:
      - "22"
      - "80"
      - "443"
    descr: "Administrative service ports"
```

#### Rules

```yaml
pfsense_firewall_rules:
  - name: "Allow LAN to TFTP on tuareg"
    interface: "lan"
    action: "pass"
    protocol: "udp"
    source: "172.23.11.0/24"
    destination: "172.23.11.5"
    destination_port: "69"

  - name: "Block guest to restricted networks"
    interface: "opt5"  # VL50_GUEST
    action: "block"
    protocol: "any"
    source: "172.23.50.0/24"
    destination: "RESTRICTED_NETWORKS"  # Can use alias

  - name: "Allow SSH to pfSense from mgmt"
    interface: "lan"
    action: "pass"
    protocol: "tcp"
    source: "172.23.11.0/24"
    destination: "(self)"
    destination_port: "22"
```

## Dependencies

None.

## Example Playbook

### Enable PXE Boot on LAN Interface

```yaml
---
- name: Configure pfSense DHCP PXE Boot
  hosts: pfsense
  gather_facts: false

  vars:
    pfsense_manage_dhcp: true
    pfsense_dhcp_pxe_config:
      interface: "lan"
      next_server: "172.23.11.5"  # PXE server (tuareg)
      use_http_boot: true
      http_boot_url: "http://172.23.11.5/boot.ipxe"
      filename_bios: "boot.ipxe"
      filename_uefi: "http://172.23.11.5/boot.ipxe"

  roles:
    - rhdc.network.pfsense
```

### Configure Static DHCP Mappings

```yaml
---
- name: Configure pfSense DHCP Static Mappings
  hosts: pfsense
  gather_facts: false

  vars:
    pfsense_manage_dhcp: true
    pfsense_dhcp_static_mappings:
      - interface: "opt11"  # VL12_SERVER
        mac: "3c:ec:ef:ba:22:c1"
        ipaddr: "172.23.12.42"
        hostname: "kvmzfs02"
        domain: "open.lab.rodhouse.net"
        descr: "kvmzfs02 - Primary 2.5G interface"

      - interface: "opt12"  # VL10_BMC
        mac: "3c:ec:ef:33:63:ce"
        ipaddr: "172.23.10.42"
        hostname: "kvmzfs02-ipmi"
        domain: "lab.rodhouse.net"
        descr: "kvmzfs02 - IPMI/BMC"

  roles:
    - rhdc.network.pfsense
```

### Update OpenVPN Server Certificates

```yaml
---
- name: Update pfSense OpenVPN Certificate
  hosts: pfsense
  gather_facts: true

  vars:
    pfsense_manage_openvpn: true
    pfsense_manage_openvpn_certificates: true
    pfsense_openvpn_server_name: "LOCAL VPN TUNNEL"
    pfsense_openvpn_certificate_name: "rodhouse-dot-net-wildcard"
    pfsense_openvpn_expiration_threshold: 30  # Update if < 30 days left

  roles:
    - rhdc.network.pfsense
```

**Note**: After certificate update, manually restart OpenVPN service via web GUI (VPN → OpenVPN → Servers → Restart openvpn service).

### Backup pfSense Configuration

```yaml
---
- name: Backup pfSense Configuration
  hosts: pfsense
  gather_facts: false

  vars:
    pfsense_manage_backup: true
    pfsense_backup_dir: "{{ playbook_dir }}/backups/pfsense"
    pfsense_backup_retention_days: 30

  roles:
    - rhdc.network.pfsense
```

## Architecture

### API vs PHP Shell Strategy

The role implements a hybrid approach:

1. **REST API First**: Attempts to use pfSense REST API when available
2. **PHP Shell Fallback**: Uses PHP shell for unsupported operations
3. **Transparent**: Users don't need to know which method is used

### Current API Support

| Feature | REST API | PHP Shell | Notes |
|---------|----------|-----------|-------|
| System Version | ✅ | ✅ | API preferred |
| HAProxy | ✅ | ✅ | API preferred |
| Firewall Aliases | ✅ | ✅ | API preferred (v2) |
| Firewall Rules | ✅ | ✅ | API preferred (v2) |
| DHCP | ❌ | ✅ | PHP only (API planned) |
| DNS | ❌ | ✅ | PHP only |

### Module Organization

```
tasks/
├── main.yml           # Role entry point
├── dhcp/
│   ├── main.yml       # DHCP module entry
│   ├── pxe_boot.yml   # PXE boot configuration
│   └── static_mapping.yml  # Static DHCP mappings
├── dns/
│   └── main.yml       # DNS management (planned)
├── firewall/
│   └── main.yml       # Firewall rules (planned)
└── haproxy/
    └── main.yml       # HAProxy management (planned)
```

## Migrating Existing Playbooks

### Before (PHP Shell)

```yaml
- name: Enable PXE boot
  ansible.builtin.raw: |
    pfSsh.php <<'EOF'
    $config = parse_config(true);
    $config['dhcpd']['lan']['next-server'] = '192.168.1.5';
    write_config("Enabled PXE");
    services_dhcpd_configure();
    EOF
```

### After (Role)

```yaml
- hosts: pfsense
  vars:
    pfsense_manage_dhcp: true
    pfsense_dhcp_pxe_config:
      interface: "lan"
      next_server: "192.168.1.5"
      use_http_boot: true
      http_boot_url: "http://192.168.1.5/boot.ipxe"
  roles:
    - rhdc.network.pfsense
```

## Development

### Adding New Modules

1. Create module directory under `tasks/` (e.g., `tasks/firewall/`)
2. Create `main.yml` entry point
3. Implement API method first (if available)
4. Implement PHP shell fallback
5. Add feature flag to `defaults/main.yml`
6. Add conditional include to `tasks/main.yml`
7. Update README

### Testing

```bash
# Test API connectivity
ansible-playbook tests/test-api-connection.yml

# Test DHCP PXE configuration
ansible-playbook tests/test-dhcp-pxe.yml

# Test static mappings
ansible-playbook tests/test-dhcp-static.yml
```

## Implementation Notes

### Firewall Management via REST API

**Status**: ✅ Fully functional via REST API v2

Firewall alias and rule management are implemented using the pfSense REST API v2, which provides reliable, idempotent configuration management.

**Authentication**: Uses `X-API-Key` header for authentication
**Endpoints**:
- `/api/v2/firewall/alias` - Alias management (GET, POST, PATCH, DELETE)
- `/api/v2/firewall/rule` - Rule management (GET, POST, PATCH, DELETE)

**Key Features**:
- Idempotent operations (check-then-create/update pattern)
- Proper error handling with status codes
- Support for all alias types (host, network, port)
- Full rule configuration (protocols, sources, destinations, ports)

See `docs/automation/pfsense-api-implementation-summary.md` for detailed implementation notes.

### DHCP Management

**Status**: ✅ Fully functional via PHP shell

DHCP PXE boot and static mapping management work correctly using PHP shell methods. Future enhancement may migrate to REST API when endpoints become available.

## License

MIT

## Author Information

RHDC Network Team - Rodhouse Datacenter
