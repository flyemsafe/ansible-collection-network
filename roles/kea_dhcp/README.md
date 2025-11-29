# rhdc.network.kea_dhcp

Deploy Kea DHCP server in Podman container with iPXE chainloading support for UEFI + BIOS PXE boot.

## Overview

This role deploys [ISC Kea DHCP](https://www.isc.org/kea/) as a containerized DHCP server with:
- **iPXE Detection**: Automatic boot file selection based on client capabilities
- **UEFI Support**: Proper chainloading from EFI → iPXE → boot menu
- **BIOS Support**: Direct boot to iPXE menu
- **JSON Configuration**: Ansible-friendly templates
- **Podman Native**: Rootful container with systemd service

## Requirements

- Red Hat Enterprise Linux 8/9 or Fedora 38+
- Podman installed
- Root/sudo access on target host
- Network access to TFTP/HTTP boot server

## Role Variables

See `defaults/main.yml` for complete list. Key variables:

```yaml
# Subnet configuration
kea_subnets:
  - subnet: "172.23.11.0/24"
    pools:
      - "172.23.11.100 - 172.23.11.200"
    gateway: "172.23.11.1"
    dns_servers:
      - "172.23.11.1"
    domain_name: "lab.rodhouse.net"
    lease_time: 86400

# PXE Boot configuration
kea_pxe_enabled: true
kea_pxe_next_server: "172.23.11.5"
kea_pxe_boot_file_bios: "boot.ipxe"
kea_pxe_boot_file_uefi: "rhdc-boot.efi"
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
