# Ansible Role: rhdc.network.pxeboot

Deploy a simple PXE boot server using dnsmasq (TFTP) + nginx (HTTP) in a containerized environment.

## Overview

This role deploys a lightweight PXE boot server designed for hosting local RHEL ISO repositories. It replaces the complex netboot.xyz setup with a purpose-built container.

**Components:**
- **dnsmasq**: TFTP server (port 69/udp) for iPXE boot files
- **nginx**: HTTP server (port 80/tcp) for ISOs, kernels, and initrd images
- **Alpine Linux**: Minimal container base (~12MB)
- **Systemd service**: Auto-start and restart on failure

## Requirements

- **OS**: RHEL 8+, Fedora 38+
- **Container Runtime**: Podman (rootful for privileged ports 69/80)
- **Systemd**: For service management
- **Storage**: Loop-mounted ISOs with SELinux context

## Role Variables

Available variables with defaults:

```yaml
# Container configuration
pxeboot_container_name: rhdc-pxeboot
pxeboot_image_name: rhdc-pxeboot
pxeboot_image_tag: latest

# Paths
pxeboot_base_dir: /opt/podman/containers/rhdc-pxeboot
pxeboot_menus_dir: "{{ pxeboot_base_dir }}/menus"
pxeboot_config_dir: "{{ pxeboot_base_dir }}/config"
pxeboot_container_dir: "{{ pxeboot_base_dir }}/container"
pxeboot_logs_dir: "{{ pxeboot_base_dir }}/logs"

# ISO repository location
pxeboot_iso_mount_base: /srv/rhel-isos

# Network
pxeboot_server_ip: "{{ ansible_default_ipv4.address }}"
pxeboot_tftp_port: 69
pxeboot_http_port: 80

# Timezone
pxeboot_timezone: America/New_York

# RHEL versions to configure
pxeboot_rhel_versions:
  - version: "8.10"
    iso_dir: rhel-8.10
  - version: "9.7"
    iso_dir: rhel-9.7
  - version: "10.1"
    iso_dir: rhel-10.1

# Menu configuration
pxeboot_menu_title: "Rodhouse Datacenter - PXE Boot Menu"
pxeboot_menu_timeout: 10000  # 10 seconds
pxeboot_default_option: rhel10
```

## Dependencies

None.

## Example Playbook

```yaml
- name: Deploy RHDC PXE Boot Server
  hosts: pxeboot_servers
  become: true

  roles:
    - role: rhdc.network.pxeboot
      vars:
        pxeboot_server_ip: 172.23.11.5
        pxeboot_rhel_versions:
          - version: "8.10"
            iso_dir: rhel-8.10
          - version: "9.7"
            iso_dir: rhel-9.7
          - version: "10.1"
            iso_dir: rhel-10.1
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
│   ├── rhel-8.ipxe             # RHEL 8 boot
│   ├── rhel-9.ipxe             # RHEL 9 boot
│   └── rhel-10.ipxe            # RHEL 10 boot
└── logs/                       # Container logs (journald)
```

## Tasks Performed

1. **Create directory structure** for container configs and menus
2. **Build container image** with Alpine + dnsmasq + nginx
3. **Generate iPXE menus** for each RHEL version
4. **Create dnsmasq configuration** for TFTP server
5. **Create nginx configuration** for HTTP server
6. **Deploy systemd service** for auto-start and monitoring
7. **Start and enable service**

## Systemd Service

The role creates `/etc/systemd/system/rhdc-pxeboot.service`:

```ini
[Unit]
Description=RHDC PXE Boot Server (dnsmasq + nginx)
After=network-online.target

[Service]
Type=simple
Restart=always
ExecStart=/usr/bin/podman run --rm --name rhdc-pxeboot ...

[Install]
WantedBy=multi-user.target
```

## Testing

After deployment, verify:

```bash
# Check service status
sudo systemctl status rhdc-pxeboot.service

# Test TFTP
echo "get boot.ipxe /tmp/test.ipxe" | tftp <server-ip>
ls -lh /tmp/test.ipxe

# Test HTTP
curl -I http://<server-ip>/rhel-10.1/images/pxeboot/vmlinuz

# PXE boot a client
# Should see: "Rodhouse Datacenter - PXE Boot Menu"
```

## Firewall Rules

Ensure firewall allows:
- **UDP 69** (TFTP) from client VLANs
- **TCP 80** (HTTP) from client VLANs

## Cleanup

To remove old netboot.xyz installation:

```bash
# Stop old service
sudo systemctl stop netbootxyz.service
sudo systemctl disable netbootxyz.service

# Remove files
sudo rm -rf /opt/podman/containers/netbootxyz
sudo rm /etc/systemd/system/netbootxyz.service
sudo systemctl daemon-reload
```

## License

MIT

## Author Information

Rodhouse Datacenter Infrastructure Team
