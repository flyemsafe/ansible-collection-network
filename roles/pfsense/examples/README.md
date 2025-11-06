# pfSense Role Examples

This directory contains example playbooks demonstrating how to use the `rhdc.network.pfsense` role.

## Examples

### 1. configure-pxe-boot.yml

**Purpose**: Configure PXE boot settings on a DHCP interface.

**Features**:
- HTTP-based iPXE boot configuration
- UEFI and BIOS boot support
- Modern netboot.xyz integration

**Use Case**: Enable network booting for bare metal server installations.

**Usage**:
```bash
ansible-playbook -i inventory/hosts configure-pxe-boot.yml
```

---

### 2. configure-static-mappings.yml

**Purpose**: Configure DHCP static mappings (IP reservations) across multiple interfaces.

**Features**:
- Multiple interface support
- Primary and BMC network interfaces
- Idempotent operation

**Use Case**: Reserve specific IP addresses for infrastructure servers and management interfaces.

**Usage**:
```bash
ansible-playbook -i inventory/hosts configure-static-mappings.yml
```

---

### 3. configure-complete-dhcp.yml

**Purpose**: Comprehensive DHCP configuration combining PXE boot and static mappings.

**Features**:
- Combined PXE + static mapping configuration
- Multi-VLAN network architecture
- Real-world datacenter setup

**Use Case**: Complete network setup for a datacenter environment with PXE boot capability and reserved IPs for infrastructure.

**Usage**:
```bash
ansible-playbook -i inventory/hosts configure-complete-dhcp.yml
```

---

## Prerequisites

Before running these examples, you need to:

1. **Configure pfSense connection variables** in your inventory or group_vars:

```yaml
# group_vars/pfsense.yml
pfsense_api_url: "https://172.23.11.1/api/v1"
pfsense_api_key: "{{ vault_pfsense_api_key }}"  # Store in Ansible Vault
pfsense_validate_certs: false
```

2. **Install the collection**:

```bash
# If using a local collection
ansible-galaxy collection install /path/to/rhdc-ansible/ansible-collection-network

# Or from requirements.yml
ansible-galaxy collection install -r requirements.yml
```

3. **Ensure pfSense is accessible**:
   - SSH access configured
   - REST API package installed (optional but recommended)
   - Admin privileges

## Customizing Examples

Each example uses variables that you can customize for your environment:

### PXE Boot Variables

```yaml
pfsense_dhcp_pxe_config:
  interface: "lan"                              # DHCP interface name
  next_server: "192.168.1.5"                    # PXE server IP
  use_http_boot: true                           # Use HTTP iPXE (true) or TFTP (false)
  http_boot_url: "http://192.168.1.5/boot.ipxe" # HTTP boot URL
  filename_bios: "boot.ipxe"                    # BIOS boot filename
  filename_uefi: "http://192.168.1.5/boot.ipxe" # UEFI boot filename/URL
```

### Static Mapping Variables

```yaml
pfsense_dhcp_static_mappings:
  - interface: "lan"                    # DHCP interface name
    mac: "00:11:22:33:44:55"            # MAC address
    ipaddr: "192.168.1.100"             # IP address to reserve
    hostname: "server01"                # Hostname
    domain: "lab.local"                 # Domain (optional)
    descr: "Production Server"          # Description (optional)
```

## Testing

After running an example:

1. **Verify PXE boot**:
   - Power on a system connected to the configured interface
   - Configure BIOS/UEFI for network boot
   - System should boot from the PXE server

2. **Verify static mappings**:
   - Reboot systems to obtain new DHCP leases
   - Check assigned IPs: `ip addr show` or `ipconfig /all`
   - Test connectivity: `ping <assigned-ip>`

3. **Check pfSense logs**:
   ```bash
   # SSH to pfSense
   ssh admin@<pfsense-ip>

   # View DHCP logs
   clog /var/log/dhcpd.log | tail -50
   ```

## Architecture

The `rhdc.network.pfsense` role uses a hybrid approach:

1. **REST API First**: Attempts to use pfSense REST API when available
2. **PHP Shell Fallback**: Uses PHP shell for operations not yet supported by the API
3. **Transparent**: Users don't need to know which method is used

Current DHCP operations use PHP shell as the REST API doesn't yet support DHCP configuration. The role is designed to automatically switch to the API when support becomes available.

## Troubleshooting

### API Connection Issues

If the role reports "API unavailable", check:

1. API package is installed on pfSense
2. API key is correct and not expired
3. pfSense is accessible from the Ansible control node
4. Firewall rules allow API access

### PHP Shell Issues

If PHP shell commands fail:

1. Check SSH connectivity: `ssh admin@<pfsense-ip>`
2. Verify interface names match pfSense configuration
3. Check DHCP is enabled on the interface
4. Review pfSense system logs

### Idempotency Issues

If the playbook reports changes on every run:

1. Check MAC address format (lowercase with colons)
2. Verify interface names are correct
3. Ensure IP addresses are within DHCP pool range

## Related Documentation

- [pfSense Role README](../README.md) - Main role documentation
- [pfSense REST API Documentation](https://docs.netgate.com/pfsense/en/latest/api/index.html)
- [netboot.xyz Documentation](https://netboot.xyz/docs/)

## Contributing

To add new examples:

1. Create a new playbook file in this directory
2. Follow the existing naming convention: `configure-<feature>.yml`
3. Include comprehensive comments and post_tasks summary
4. Update this README with example description
5. Test the example in a lab environment
