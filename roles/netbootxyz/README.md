# rhdc.network.netbootxyz

Deploy and customize netboot.xyz PXE boot server with RHDC-specific menus and branding.

## Description

This role automates the deployment and customization of netboot.xyz, a network boot utility that provides a centralized PXE boot server for multiple operating systems. The role supports:

- Container deployment via Podman with systemd management
- Custom branding (site name, boot timeout)
- RHDC-specific custom menus (OpenShift, ARM/Raspberry Pi)
- Firewall configuration
- Template-based customization
- Idempotent operations

## Requirements

- **Operating System:** Fedora 38+ or RHEL 9+
- **Podman:** Installed and configured
- **Storage:** `/opt/podman` filesystem mounted (for servers)
- **Root Access:** Required for root container deployment
- **Firewall:** firewalld installed and running (if firewall configuration enabled)

## Role Variables

### Container Configuration

```yaml
netbootxyz_image: "ghcr.io/netbootxyz/netbootxyz:latest"
netbootxyz_config_dir: "/opt/podman/containers/netbootxyz/config"
netbootxyz_assets_dir: "/opt/podman/containers/netbootxyz/assets"
netbootxyz_timezone: "America/New_York"
```

### Port Configuration

```yaml
netbootxyz_web_port: 3000        # Web UI
netbootxyz_http_port: 80         # HTTP boot
netbootxyz_tftp_port: 69         # TFTP
```

### Branding

```yaml
netbootxyz_site_name: "Rodhouse Datacenter"
netbootxyz_boot_timeout: 60000  # milliseconds (60 seconds)
```

### Custom Menus

```yaml
netbootxyz_enable_custom_menus: true
netbootxyz_enable_openshift_menu: true
netbootxyz_enable_arm_menu: true
netbootxyz_enable_simplified_main: false  # Opt-in
```

### Firewall Configuration

```yaml
netbootxyz_configure_firewall: true
netbootxyz_firewall_zone: "public"
netbootxyz_firewall_ports:
  - "{{ netbootxyz_web_port }}/tcp"
  - "{{ netbootxyz_http_port }}/tcp"
  - "{{ netbootxyz_tftp_port }}/udp"
```

### Container Behavior

```yaml
netbootxyz_use_host_networking: true
netbootxyz_auto_restart: true
netbootxyz_restart_policy: "always"
```

### Service Management

```yaml
netbootxyz_service_name: "netbootxyz.service"
netbootxyz_service_enabled: true
netbootxyz_service_started: true
```

### OpenShift/CoreOS Configuration

```yaml
netbootxyz_coreos_stable_version: "42.20250721.3.0"
netbootxyz_coreos_testing_version: "42.20250803.2.0"
netbootxyz_coreos_next_version: "42.20250803.1.0"
netbootxyz_default_ignition_url: ""  # Optional
```

See `defaults/main.yml` for the complete list of variables.

## Dependencies

- `ansible.posix` collection (for firewalld module)
- `containers.podman` collection (for podman_image module)

## Example Playbook

### Basic Deployment with Defaults

```yaml
---
- name: Deploy netboot.xyz
  hosts: pxe_servers
  become: true

  roles:
    - role: rhdc.network.netbootxyz
```

### Custom Branding and Menus

```yaml
---
- name: Deploy netboot.xyz with RHDC Customization
  hosts: tuareg.lab.rodhouse.net
  become: true

  vars:
    netbootxyz_site_name: "RHDC"
    netbootxyz_boot_timeout: 30000  # 30 seconds
    netbootxyz_enable_openshift_menu: true
    netbootxyz_enable_arm_menu: true
    netbootxyz_enable_simplified_main: false

  roles:
    - role: rhdc.network.netbootxyz
```

### Deploy Only (No Customization)

```yaml
---
- name: Deploy netboot.xyz container only
  hosts: pxe_servers
  become: true

  vars:
    netbootxyz_enable_custom_menus: false

  roles:
    - role: rhdc.network.netbootxyz
      tags: ['deploy']
```

### Customize Only (No Container Redeployment)

```yaml
---
- name: Update netboot.xyz customization
  hosts: pxe_servers
  become: true

  vars:
    netbootxyz_site_name: "Updated Site Name"

  roles:
    - role: rhdc.network.netbootxyz
      tags: ['customize']
```

## Tags

The role supports the following tags for selective execution:

- `netbootxyz` - Run all tasks
- `deploy` - Deploy container only
- `firewall` - Configure firewall only
- `customize` - Apply customization only

### Tag Usage Examples

```bash
# Full deployment
ansible-playbook playbook.yml

# Deploy container only (no customization)
ansible-playbook playbook.yml --tags deploy

# Customize only (no container redeployment)
ansible-playbook playbook.yml --tags customize

# Configure firewall only
ansible-playbook playbook.yml --tags firewall

# Deploy and configure firewall (skip customization)
ansible-playbook playbook.yml --tags deploy,firewall
```

## Custom Menus

### OpenShift/CoreOS Menu

Enabled by default with `netbootxyz_enable_openshift_menu: true`.

Provides:
- Fedora CoreOS Stable/Testing/Next
- OpenShift Single Node (SNO) installer support
- OpenShift IPI installer support
- Interactive CoreOS ignition URL configuration

### ARM/Raspberry Pi Menu

Enabled by default with `netbootxyz_enable_arm_menu: true`.

Provides:
- Fedora ARM64 Server
- Raspberry Pi OS
- Home Assistant OS
- Ubuntu Server ARM64

### Simplified Main Menu

Opt-in with `netbootxyz_enable_simplified_main: true`.

Replaces the default netboot.xyz main menu with a streamlined RHDC-specific menu showing only:
- RHEL
- Fedora
- Fedora CoreOS
- Windows
- Custom RHDC menus
- Utilities

## Access Points

After deployment, access netboot.xyz via:

- **Web UI:** `http://<hostname>:3000`
- **HTTP Boot:** `http://<hostname>:80/boot.ipxe`
- **TFTP Boot:** `<hostname>:69`

## Verification

```bash
# Check service status
sudo systemctl status netbootxyz.service

# View logs
sudo podman logs -f netbootxyz

# Test HTTP access
curl http://<hostname>:80/boot.ipxe

# Verify firewall
sudo firewall-cmd --list-ports --zone=public
```

## File Locations

- **Config:** `{{ netbootxyz_config_dir }}`
- **Assets:** `{{ netbootxyz_assets_dir }}`
- **Custom Menus:** `{{ netbootxyz_config_dir }}/menus/`
- **Systemd Unit:** `/etc/systemd/system/netbootxyz.service`

## Idempotency

The role is fully idempotent:
- Systemd unit only updated if template changes
- Service only restarted if configuration changes
- Firewall rules only added if not present
- Custom menus only updated if content changes

## License

MIT

## Author Information

Rodhouse Datacenter Infrastructure Team

## See Also

- [netboot.xyz Documentation](https://netboot.xyz/docs/)
- [netboot.xyz GitHub](https://github.com/netbootxyz/netbootxyz)
- [iPXE Documentation](https://ipxe.org/)
