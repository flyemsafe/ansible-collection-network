# rhdc.network.kea_dhcp

Deploy Kea DHCP server in Podman container with iPXE chainloading support for UEFI + BIOS PXE boot.

## Overview

This role deploys [ISC Kea DHCP](https://www.isc.org/kea/) as a containerized DHCP server with:
- **iPXE Detection**: Automatic boot file selection based on client capabilities
- **UEFI Support**: Proper chainloading from EFI → iPXE → boot menu
- **BIOS Support**: Direct boot to iPXE menu
- **JSON Configuration**: Ansible-friendly templates
- **Pre-built Image**: Uses [jonasal/kea-dhcp4](https://github.com/JonasAlfredsson/docker-kea) (no build on every run)
- **Quadlet Deployment**: Modern systemd-integrated container management

## Requirements

- Red Hat Enterprise Linux 8/9 or Fedora 38+
- Podman installed
- Root/sudo access on target host
- Network access to TFTP/HTTP boot server

## Role Variables

See `defaults/main.yml` for complete list. Key variables:

```yaml
# Network interface configuration
kea_interface: "qubibr0"  # Specific interface (recommended)
                           # Default: "*" (all interfaces - may cause conflicts)

# Subnet configuration with per-subnet reservations
kea_subnets:
  - subnet: "172.23.11.0/24"
    pools:
      - "172.23.11.100 - 172.23.11.200"
    gateway: "172.23.11.1"
    dns_servers:
      - "172.23.11.1"
    ntp_servers:
      - "172.23.11.1"
    domain_name: "lab.rodhouse.net"
    lease_time: 86400
    # Per-subnet static reservations (optional)
    reservations:
      - hostname: "server01"
        hw_address: "aa:bb:cc:dd:ee:ff"
        ip_address: "172.23.11.10"
        description: "Production web server"  # Optional, stored as Kea comment

# PXE Boot configuration
kea_pxe_enabled: true
kea_pxe_next_server: "172.23.11.5"
kea_pxe_boot_file_bios: "boot.ipxe"
kea_pxe_boot_file_uefi: "rhdc-boot.efi"
```

### Static Reservations

Reservations are defined per-subnet using the `reservations` list:

| Field | Required | Description |
|-------|----------|-------------|
| `hostname` | Yes | Device hostname |
| `hw_address` | Yes | MAC address (format: aa:bb:cc:dd:ee:ff) |
| `ip_address` | Yes | Static IP to assign |
| `description` | No | Optional comment stored in Kea config |

**Example with multiple reservations**:
```yaml
kea_subnets:
  - subnet: "172.23.11.0/24"
    # ... other config ...
    reservations:
      - hostname: "switch01"
        hw_address: "1c:5f:2b:27:cc:00"
        ip_address: "172.23.11.2"
        description: "Core switch"
      - hostname: "server01"
        hw_address: "3c:ec:ef:34:ca:39"
        ip_address: "172.23.11.43"
        description: "KVM/ZFS host"
```

### Important: Network Interface Configuration

**⚠️ CRITICAL**: Always specify `kea_interface` to avoid conflicts with other DHCP servers.

**Why?** The default `kea_interface: "*"` causes Kea to attempt binding to ALL network interfaces, which can conflict with:
- Libvirt's default network (`virbr0`) DHCP
- Other DHCP servers on the same host
- Docker/Podman bridge networks

**Recommended**: Specify the exact bridge/interface name:
```yaml
kea_interface: "qubibr0"  # Your VLAN bridge interface
```

## Dependencies

None.

## Example Playbook

```yaml
---
- name: Deploy Kea DHCP Server
  hosts: tuareg.lab.rodhouse.net
  become: true
  gather_facts: true
  
  roles:
    - role: rhdc.network.kea_dhcp
      kea_subnets:
        - subnet: "172.23.11.0/24"
          pools:
            - "172.23.11.100 - 172.23.11.200"
          gateway: "172.23.11.1"
          dns_servers:
            - "172.23.11.1"
          domain_name: "lab.rodhouse.net"
          lease_time: 86400
```

## Troubleshooting

### Port 67 Conflict ("Address in use")

**Symptom**: Kea logs show:
```
DHCPSRV_OPEN_SOCKET_FAIL failed to open socket: Failed to open socket on interface X, reason: failed to bind fallback socket to address Y, port 67, reason: Address in use
```

**Cause**: Another DHCP server is using port 67 on the same interface.

**Common Conflicts**:
1. **Libvirt default network**:
   ```bash
   sudo virsh net-destroy default
   sudo virsh net-autostart default --disable
   ```

2. **pfSense/OPNsense DHCP**: Disable DHCP on the VLAN interface via web UI

3. **Other DHCP servers**: Check with `sudo ss -ulnp 'sport = :67'`

**Prevention**: Always specify `kea_interface` to bind to a specific interface instead of `"*"`.

### Verifying Kea is Running

```bash
# Check service status
sudo systemctl status rhdc-kea-dhcp.service

# View DHCP transactions
sudo podman logs -f rhdc-kea-dhcp

# Check active leases
sudo podman exec rhdc-kea-dhcp cat /kea/leases/kea-leases4.csv

# Verify port 67 binding
sudo ss -ulnp 'sport = :67'

# Check Quadlet unit
cat /etc/containers/systemd/rhdc-kea-dhcp.container
```

## iPXE Boot Flow

**UEFI Client:**
1. Client → DHCP Request (UEFI firmware)
2. Kea detects architecture → responds with `rhdc-boot.efi`
3. Client downloads `rhdc-boot.efi` via TFTP, executes iPXE firmware
4. iPXE → DHCP Request (user-class: "iPXE")
5. Kea detects iPXE → responds with `boot.ipxe`
6. iPXE downloads `boot.ipxe`, shows PXE menu

**BIOS Client:**
1. Client → DHCP Request (BIOS PXE)
2. Kea responds with `boot.ipxe`
3. Client downloads and executes directly

## License

MIT

## Author

RHDC Infrastructure Team
