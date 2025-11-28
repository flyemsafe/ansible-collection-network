# Traefik Role

Deploy and manage Traefik reverse proxy with Podman and Quadlet systemd integration.

## Description

This role deploys Traefik as a rootless Podman container managed by systemd Quadlet. It supports:

- ✅ Multi-host routing (backends on any server)
- ✅ ACME/Let's Encrypt with DNS-01 challenge (Cloudflare)
- ✅ File Provider for hot-reload configuration
- ✅ Built-in dashboard and metrics
- ✅ Health checks and monitoring
- ✅ Custom middlewares (headers, rate limiting, auth, etc.)

## Requirements

- **OS**: Fedora 38+, RHEL 9+
- **Podman**: 4.0+
- **Python**: 3.9+
- **Ansible**: 2.15+
- **Collections**: `containers.podman`

## Role Variables

### Basic Configuration

```yaml
traefik_version: "v3.2"                    # Traefik version
traefik_host: "{{ inventory_hostname }}"   # Deployment host
traefik_user: "{{ ansible_user }}"         # User running container
traefik_container_name: "traefik"          # Container name
```

### Storage Paths

```yaml
traefik_base_dir: "/opt/podman/containers/traefik"
traefik_config_dir: "{{ traefik_base_dir }}/config"
traefik_dynamic_config_dir: "{{ traefik_config_dir }}/dynamic"
traefik_logs_dir: "{{ traefik_base_dir }}/logs"
```

### Network Configuration

```yaml
traefik_http_port: 80       # HTTP port (redirects to HTTPS)
traefik_https_port: 443     # HTTPS port
traefik_dashboard_port: 8080  # Dashboard and API port
```

### ACME / Let's Encrypt

```yaml
traefik_acme_enabled: true
traefik_acme_email: "admin@example.com"
traefik_acme_challenge_type: "dns-01"
traefik_acme_dns_provider: "cloudflare"
traefik_cloudflare_email: "{{ vault_cloudflare_email }}"
traefik_cloudflare_api_token: "{{ vault_cloudflare_api_token }}"
```

### Service Routing

Define services in inventory (group_vars or host_vars):

```yaml
traefik_services:
  - name: unifi
    hostname: unifi.lab.rodhouse.net
    backend_url: https://tuareg.lab.rodhouse.net:8443
    insecure_skip_verify: true
    health_check_path: /
    middlewares:
      - headers-forwarded-proto

  - name: inventory
    hostname: inventory.lab.rodhouse.net
    backend_url: http://tuareg.lab.rodhouse.net:8081
    health_check_path: /api/status
```

### Middlewares

```yaml
traefik_middlewares:
  - name: security-headers
    type: headers
    config:
      customResponseHeaders:
        X-Frame-Options: "SAMEORIGIN"
        X-Content-Type-Options: "nosniff"

  - name: rate-limit-api
    type: ratelimit
    config:
      average: 100
      burst: 50
      period: 1m
```

## Dependencies

None

## Example Playbook

```yaml
---
- name: Deploy Traefik Reverse Proxy
  hosts: tuareg.lab.rodhouse.net
  become: false

  roles:
    - role: rhdc.network.traefik
      vars:
        traefik_acme_enabled: true
        traefik_cloudflare_email: "{{ vault_cloudflare_email }}"
        traefik_cloudflare_api_token: "{{ vault_cloudflare_api_token }}"
        traefik_services:
          - name: unifi
            hostname: unifi.lab.rodhouse.net
            backend_url: https://tuareg.lab.rodhouse.net:8443
            insecure_skip_verify: true
            middlewares:
              - headers-forwarded-proto
```

## Testing

```bash
# Deploy to test host
ansible-playbook -i inventory/hosts playbooks/infrastructure/deploy-traefik.yml --check

# Deploy for real
ansible-playbook -i inventory/hosts playbooks/infrastructure/deploy-traefik.yml

# Validate deployment
ansible-playbook -i inventory/hosts playbooks/infrastructure/deploy-traefik.yml --tags traefik-validate

# Check Traefik dashboard
curl -k http://tuareg.lab.rodhouse.net:8080/dashboard/
```

## Troubleshooting

### Container not starting

```bash
# Check container logs
podman logs traefik

# Check systemd status
systemctl --user status traefik

# Validate configuration
podman run --rm -v /opt/podman/containers/traefik/config:/config:ro \
  traefik:v3.2 --configFile=/config/traefik.yml --validate
```

### ACME certificate issues

```bash
# Check ACME storage
ls -lh /opt/podman/containers/traefik/config/acme.json

# Check Traefik logs for ACME errors
podman logs traefik | grep -i acme

# Verify Cloudflare API token permissions
curl -X GET "https://api.cloudflare.com/client/v4/user/tokens/verify" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Service not routing

```bash
# Check dynamic configuration
cat /opt/podman/containers/traefik/config/dynamic/services.yml

# Verify Traefik sees the service
curl http://localhost:8080/api/http/routers

# Check backend health
curl http://localhost:8080/api/http/services
```

## Maintenance

### Adding a new service

1. Add service to `traefik_services` in inventory
2. Re-run playbook (config hot-reloads automatically)

### Updating Traefik version

1. Update `traefik_version` variable
2. Re-run playbook (pulls new image and restarts)

### Certificate renewal

Automatic via ACME. Check status:

```bash
podman logs traefik | grep -i "renew"
```

## License

MIT

## Author

RHDC Infrastructure Team
