# Ansible Role: rhdc.network.pxeboot

Deploy a simple PXE boot server using dnsmasq (TFTP) + nginx (HTTP) in a containerized environment.

## Overview

This role deploys a lightweight PXE boot server designed for hosting local RHEL ISO repositories. It replaced the previous netboot.xyz deployment with a purpose-built, simpler solution.

### Why rhdc-pxeboot Instead of netboot.xyz?

| Issue with netboot.xyz | rhdc-pxeboot Solution |
|------------------------|----------------------|
| 100+ OS options (only needed RHEL) | Focused on RHEL only |
| Complex customization required | Simple, purpose-built menus |
| ~500MB container image | ~12MB Alpine container |
| Configuration regenerated on restart | Persistent volume-mounted config |
| Difficult to brand/customize | Native RHDC branding |
| Tracking upstream security updates | Self-maintained, minimal surface |

**Decision**: Replaced netboot.xyz in November 2025 (Issue #319) for simplicity and maintainability in homelab environment.

## Components

- **dnsmasq**: TFTP server (port 69/udp) for iPXE boot files
- **nginx**: HTTP server (port 80/tcp) for ISOs, kernels, and initrd images
- **Alpine Linux**: Minimal container base (~12MB)
- **Systemd service**: Auto-start and restart on failure

## Requirements

- **OS**: RHEL 8+, Fedora 38+
- **Container Runtime**: Podman (rootful for privileged ports 69/80)
- **Systemd**: For service management
- **Storage**: Loop-mounted ISOs at `/mnt/ssd/pxeboot/repos/`
- **DHCP**: Kea DHCP (VLAN 11) or pfSense (other VLANs) configured with PXE options

## Task Files

| Task File | Purpose | Tag |
|-----------|---------|-----|
| `main.yml` | Deploy PXE boot server | `pxeboot` |
| `test.yml` | Test TFTP/HTTP connectivity | `pxeboot_test` |
| `verify.yml` | Verify infrastructure status | `pxeboot_verify` |
| `create_test_vm.yml` | Create test VM for PXE verification | `pxeboot_test_vm` |
| `cleanup_test_vm.yml` | Remove test VM | `pxeboot_cleanup_vm` |

## Role Variables

### Container Configuration

```yaml
pxeboot_container_name: rhdc-pxeboot
pxeboot_image_name: rhdc-pxeboot
pxeboot_image_tag: latest
```

### Paths

```yaml
pxeboot_base_dir: /opt/podman/containers/rhdc-pxeboot
pxeboot_menus_dir: "{{ pxeboot_base_dir }}/menus"
pxeboot_config_dir: "{{ pxeboot_base_dir }}/config"
pxeboot_container_dir: "{{ pxeboot_base_dir }}/container"
pxeboot_logs_dir: "{{ pxeboot_base_dir }}/logs"
pxeboot_iso_mount_base: /mnt/ssd/pxeboot/repos
```

### Network

```yaml
pxeboot_server_ip: "{{ ansible_default_ipv4.address }}"
pxeboot_tftp_port: 69
pxeboot_http_port: 80
```

### RHEL Versions

```yaml
pxeboot_rhel_versions:
  - version: "8.10"
    iso_dir: rhel-8.10
    kernel_path: images/pxeboot/vmlinuz
    initrd_path: images/pxeboot/initrd.img
  - version: "9.7"
    iso_dir: rhel-9.7
    kernel_path: images/pxeboot/vmlinuz
    initrd_path: images/pxeboot/initrd.img
  - version: "10.1"
    iso_dir: rhel-10.1
    kernel_path: images/pxeboot/vmlinuz
    initrd_path: images/pxeboot/initrd.img
```

### Menu Configuration

```yaml
pxeboot_menu_title: "Rodhouse Datacenter - PXE Boot Menu"
pxeboot_menu_timeout: 10000  # milliseconds
pxeboot_default_option: rhel10
```

### Test VM Configuration

```yaml
pxeboot_test_vm_host: kvmzfs01          # Required for test VM tasks
pxeboot_test_vm_name: pxe-test-vm
pxeboot_test_vm_ram: 2048
pxeboot_test_vm_vcpus: 2
pxeboot_test_vm_disk: 10G
pxeboot_test_vm_network: qubibr0
```

## Example Playbooks

### Deploy PXE Boot Server

```yaml
- name: Deploy RHDC PXE Boot Server
  hosts: tuareg.lab.rodhouse.net
  become: true

  roles:
    - role: rhdc.network.pxeboot
      vars:
        pxeboot_server_ip: 172.23.11.5
```

### Test PXE Boot Infrastructure

```yaml
- name: Test PXE Boot Server
  hosts: tuareg.lab.rodhouse.net
  become: true

  tasks:
    - name: Include pxeboot test tasks
      ansible.builtin.include_role:
        name: rhdc.network.pxeboot
        tasks_from: test
      vars:
        pxeboot_server_ip: 172.23.11.5
```

### Verify Infrastructure

```yaml
- name: Verify PXE Boot Infrastructure
  hosts: tuareg.lab.rodhouse.net
  become: true

  tasks:
    - name: Include pxeboot verify tasks
      ansible.builtin.include_role:
        name: rhdc.network.pxeboot
        tasks_from: verify
```

### Create Test VM

```yaml
- name: Create PXE Test VM
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Create test VM on kvmzfs01
      ansible.builtin.include_role:
        name: rhdc.network.pxeboot
        tasks_from: create_test_vm
      vars:
        pxeboot_test_vm_host: kvmzfs01
        pxeboot_server_ip: 172.23.11.5
```

### Cleanup Test VM

```yaml
- name: Cleanup PXE Test VM
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Remove test VM from kvmzfs01
      ansible.builtin.include_role:
        name: rhdc.network.pxeboot
        tasks_from: cleanup_test_vm
      vars:
        pxeboot_test_vm_host: kvmzfs01
```

## Directory Structure

```
/opt/podman/containers/rhdc-pxeboot/
├── container/
│   ├── Containerfile           # Container build definition
│   └── start-pxeboot.sh        # Container startup script
├── config/
│   ├── dnsmasq.conf            # TFTP configuration
│   └── nginx-default.conf      # HTTP configuration
├── menus/
│   ├── boot.ipxe               # Main PXE menu
│   ├── rhel-810.ipxe           # RHEL 8.10 boot
│   ├── rhel-97.ipxe            # RHEL 9.7 boot
│   └── rhel-101.ipxe           # RHEL 10.1 boot
└── logs/                       # Container logs (journald)
```

## DHCP Configuration

### Kea DHCP (VLAN 11 - Recommended)

PXE boot options are configured in Kea DHCP on tuareg for VLAN 11 (172.23.11.0/24).

### pfSense (Other VLANs)

For VLANs still using pfSense DHCP:

1. Navigate to **Services > DHCP Server > [Interface]**
2. Scroll to **Network Booting**
3. Configure:
   - **Enable**: ✓
   - **Next Server**: `172.23.11.5`
   - **Default BIOS file name**: `boot.ipxe`
   - **UEFI 64 bit file name**: `boot.ipxe`

## Firewall Rules

Ensure firewall allows from PXE client VLANs:
- **UDP 69** (TFTP) to tuareg
- **TCP 80** (HTTP) to tuareg

## Dependencies

None.

## License

MIT

## Author Information

Rodhouse Datacenter Infrastructure Team

## Related Documentation

- `docs/infrastructure/network/pxeboot/` - Complete PXE boot documentation
- Issue #319 - netboot.xyz replacement decision
