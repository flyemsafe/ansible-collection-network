# Ansible Collection - rhdc.network

Network infrastructure automation for Rodhouse Datacenter.

## Description

This collection provides roles and modules for managing network infrastructure in Rodhouse Datacenter, including:

- **netboot.xyz** - PXE boot server deployment and customization
- **pfSense** - Firewall, router, DHCP, DNS, and HAProxy automation (planned)

## Requirements

- Ansible 2.14 or later
- Python 3.9 or later on managed nodes
- Collections:
  - ansible.posix >= 1.5.0
  - community.general >= 7.0.0
  - pfsensible.core >= 0.5.0 (for pfSense roles)

## Installation

### From Source

```bash
# Clone rhdc-ansible repository
git clone https://github.com/flyemsafe/rhdc-ansible.git
cd rhdc-ansible/ansible-collection-network

# Build and install collection
ansible-galaxy collection build
ansible-galaxy collection install rhdc-network-*.tar.gz
```

### From requirements file

```yaml
# requirements.yml
collections:
  - name: rhdc.network
    source: https://github.com/flyemsafe/rhdc-ansible
```

```bash
ansible-galaxy collection install -r requirements.yml
```

## Roles

### netbootxyz

Deploy and customize netboot.xyz PXE boot server.

**Features:**
- Podman container deployment with systemd service
- Custom branding and site name
- OpenShift/CoreOS custom menus
- ARM/Raspberry Pi custom menus
- Simplified main menu option
- Firewalld configuration

**Quick Start:**

```yaml
- name: Deploy netboot.xyz
  hosts: pxe_server
  become: true

  vars:
    netbootxyz_site_name: "My Datacenter"
    netbootxyz_enable_openshift_menu: true

  roles:
    - rhdc.network.netbootxyz
```

See [roles/netbootxyz/README.md](roles/netbootxyz/README.md) for detailed documentation.

### pfSense Roles (Planned)

Future roles for pfSense automation:
- `pfsense_dhcp` - DHCP configuration
- `pfsense_firewall` - Firewall rules and NAT
- `pfsense_dns` - DNS forwarding and IdM integration
- `pfsense_certificates` - ACME certificates
- `pfsense_backup` - Backup and restore
- `pfsense_haproxy` - HAProxy load balancer

See [Issue #199](https://github.com/flyemsafe/rodhouse-datacenter/issues/199) for details.

## Usage Examples

### Deploy netboot.xyz with custom branding

```yaml
- name: Deploy netboot.xyz with RHDC branding
  hosts: tuareg.lab.rodhouse.net
  become: true

  vars:
    netbootxyz_site_name: "Rodhouse Datacenter"
    netbootxyz_boot_timeout: 60000  # 60 seconds
    netbootxyz_enable_openshift_menu: true
    netbootxyz_enable_arm_menu: true

  roles:
    - role: rhdc.network.netbootxyz
      tags: ['netbootxyz']
```

### Customize only (no redeployment)

```bash
ansible-playbook deploy-netbootxyz.yml --tags customize
```

### Deploy without firewall configuration

```yaml
- hosts: pxe_server
  become: true

  vars:
    netbootxyz_configure_firewall: false

  roles:
    - rhdc.network.netbootxyz
```

## Testing

```bash
# Install test dependencies
pip install molecule ansible-lint yamllint

# Run tests for a specific role
cd roles/netbootxyz
molecule test
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Run tests and linters
5. Submit a pull request

## License

MIT

## Author Information

Rodhouse Datacenter Infrastructure Team

- Repository: https://github.com/flyemsafe/rhdc-ansible
- Issues: https://github.com/flyemsafe/rodhouse-datacenter/issues
- Documentation: https://github.com/flyemsafe/rodhouse-datacenter/tree/main/docs
