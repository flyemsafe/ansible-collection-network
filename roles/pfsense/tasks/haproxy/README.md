---
migration:
  status: new
  migrated_date: 2025-01-19
  migrated_by: claude
  notes: |
    HAProxy automation implementation for Issue #268.
    Provides HTTPS reverse proxy with ACME certificate automation.
---

# HAProxy Management Tasks

## Overview

This module provides automated HAProxy configuration via pfSense REST API, including:

- **Backend Management**: Configure server pools with health checks
- **Frontend Management**: Configure listening addresses, SSL, and routing (ACLs)
- **ACME Integration**: Automatic Let's Encrypt certificate issuance and renewal
- **Idempotent Operations**: Safe to run repeatedly, only applies changes when needed

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ HAProxy Management Flow                                     │
└─────────────────────────────────────────────────────────────┘

1. ACME Certificate Management (acme.yml)
   ├── Register ACME account (if needed)
   ├── For each certificate (acme_certificate.yml):
   │   ├── Check if certificate exists (GET)
   │   ├── Create/update certificate config (POST/PUT)
   │   └── Issue certificate via ACME challenge

2. Backend Configuration (backend.yml)
   ├── For each backend:
   │   ├── Check if backend exists (GET)
   │   ├── Create/update backend (POST/PUT)
   │   └── Configure health checks

3. Frontend Configuration (frontend.yml)
   ├── For each frontend:
   │   ├── Validate SSL certificate (if SSL enabled)
   │   ├── Check if frontend exists (GET)
   │   ├── Create/update frontend with ACLs (POST/PUT)
   │   └── Configure routing actions

4. Apply Configuration (apply.yml)
   └── Reload HAProxy service (POST /apply)
```

## Task Files

### main.yml
Entry point for HAProxy management. Orchestrates the workflow:
- Validates API availability
- Includes ACME tasks (if certificates defined)
- Includes backend tasks (for each backend)
- Includes frontend tasks (for each frontend)
- Applies configuration changes

### acme.yml
Manages ACME account and certificates:
- Registers ACME account (Let's Encrypt)
- Calls `acme_certificate.yml` for each certificate

### acme_certificate.yml
Configures individual ACME certificates:
- Creates certificate configuration
- Issues certificates via HTTP-01 or DNS-01 challenge
- Supports wildcards (DNS-01) and single domains (HTTP-01)

### backend.yml
Configures HAProxy backends:
- Server pools with load balancing (roundrobin, source, leastconn)
- Health checks (HTTP, SSL, TCP)
- Connection and server timeouts
- Idempotent create/update operations

### frontend.yml
Configures HAProxy frontends:
- Listening addresses and ports
- SSL/TLS configuration with certificates
- ACL definitions for routing
- Backend routing actions

### apply.yml
Applies HAProxy configuration:
- Validates configuration
- Reloads HAProxy service
- Activates all changes

## Standard RHDC Frontend

**IMPORTANT**: All RHDC applications should use the shared frontend: `rhdc_https_frontend`

This frontend already exists on pfSense and handles all lab applications:
- **Name**: `rhdc_https_frontend`
- **Description**: "Rod House Data Center Shared HTTPS frontend for all lab applications"
- **Binding**: 0.0.0.0:443 (HTTPS)
- **Current Routes**:
  - unifi.lab.rodhouse.net → unifi_backend
  - hausa.lab.rodhouse.net → pfsense_gui_backend

When adding new applications, you only need to:
1. Create a backend for your app
2. Add ACLs and actions to the existing `rhdc_https_frontend`
3. Issue an ACME certificate (if needed)

## Usage Example

### Basic Container App with HTTPS

```yaml
- name: Deploy HAProxy for my-app
  hosts: hausa
  gather_facts: false

  vars:
    pfsense_acme_email: "admin@example.com"
    pfsense_manage_haproxy: true

    # Backend configuration
    pfsense_haproxy_backends:
      - name: "my_app_backend"
        mode: "http"
        balance: "roundrobin"
        servers:
          - name: "server1"
            address: "172.23.11.10"
            port: 8080

    # Frontend configuration - Use the shared RHDC frontend
    pfsense_haproxy_frontends:
      - name: "rhdc_https_frontend"  # ← Always use this shared frontend
        bind: "0.0.0.0:443"
        ssl: true
        certificate: "my_app_cert"
        acls:
          - name: "is_my_app"
            expression: "hdr(host)"
            value: "my-app.lab.rodhouse.net"
        actions:
          - condition: "is_my_app"
            backend: "my_app_backend"

    # NO ACME certificate needed - use existing wildcard
    # The rhdc_https_frontend already has rodhouse-dot-net-wildcard configured

  roles:
    - role: rhdc.network.pfsense
```

**Note**: Certificate `my_app_cert` is NOT created because `my-app.lab.rodhouse.net` is covered by the existing wildcard `*.lab.rodhouse.net`.

### Shared HTTPS Frontend with Multiple Apps

```yaml
vars:
  # Multiple backends
  pfsense_haproxy_backends:
    - name: "app1_backend"
      servers:
        - name: "app1"
          address: "172.23.11.10"
          port: 8080

    - name: "app2_backend"
      servers:
        - name: "app2"
          address: "172.23.11.11"
          port: 9000

  # Shared frontend with ACL routing
  pfsense_haproxy_frontends:
    - name: "https_frontend"
      bind: "0.0.0.0:443"
      ssl: true
      certificate: "wildcard_cert"
      acls:
        - name: "is_app1"
          expression: "hdr(host)"
          value: "app1.lab.rodhouse.net"
        - name: "is_app2"
          expression: "hdr(host)"
          value: "app2.lab.rodhouse.net"
      actions:
        - condition: "is_app1"
          backend: "app1_backend"
        - condition: "is_app2"
          backend: "app2_backend"

  # Wildcard certificate
  pfsense_acme_certificates:
    - name: "wildcard_cert"
      domains:
        - domain: "*.lab.rodhouse.net"
          method: "dns01"  # Requires DNS provider
```

## Variables

### Required Variables

| Variable | Description |
|----------|-------------|
| `pfsense_api_url` | pfSense REST API URL |
| `pfsense_api_key` | pfSense API key (store in vault) |
| `pfsense_manage_haproxy` | Enable HAProxy management (default: false) |

### ACME Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `pfsense_acme_account` | `letsencrypt_production` | ACME account name |
| `pfsense_acme_email` | (required) | Email for ACME registration |
| `pfsense_acme_certificates` | `[]` | List of certificates to manage |

### HAProxy Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `pfsense_haproxy_backends` | `[]` | List of backend configurations |
| `pfsense_haproxy_frontends` | `[]` | List of frontend configurations |

See `defaults/main.yml` for complete variable documentation and examples.

## API Endpoints Used

```
# ACME Management
GET    /api/v2/services/acme/account
POST   /api/v2/services/acme/account_key/register
GET    /api/v2/services/acme/certificate
POST   /api/v2/services/acme/certificate
PUT    /api/v2/services/acme/certificate/{id}
POST   /api/v2/services/acme/certificate/issue

# HAProxy Management
GET    /api/v2/services/haproxy/backend
POST   /api/v2/services/haproxy/backend
PUT    /api/v2/services/haproxy/backend/{id}
GET    /api/v2/services/haproxy/frontend
POST   /api/v2/services/haproxy/frontend
PUT    /api/v2/services/haproxy/frontend/{id}
POST   /api/v2/services/haproxy/apply
```

## Prerequisites

1. **pfSense REST API** must be installed:
   ```bash
   # See: docs/automation/pfsense-haproxy-acme-automation.md
   ./scripts/install_pfsense_api.sh
   ```

2. **API Key** configured in vault:
   ```yaml
   # inventory/group_vars/all/vault.yml
   vault_pfsense_api_key: "your-api-key-here"
   ```

3. **DNS Records** configured:
   - Point application hostnames to pfSense WAN IP
   - Required for ACME HTTP-01 challenges

4. **Firewall Rules**:
   - Port 80 open (for HTTP-01 ACME challenges)
   - Port 443 open (for HTTPS traffic)

## Idempotency

All tasks are idempotent and safe to run repeatedly:

- **Check before modify**: Each task checks if resource exists before creating
- **Update when needed**: Existing resources are updated (PUT) instead of duplicated
- **Skip when current**: No changes made if configuration already matches

## Error Handling

The tasks will fail if:
- pfSense REST API is not available
- API key is invalid or missing
- SSL is enabled but no certificate specified
- ACME account email not provided for new accounts
- Backend or frontend configuration is invalid

## Related Documentation

- [pfSense HAProxy & ACME Automation](../../../../../rodhouse-datacenter/docs/automation/pfsense-haproxy-acme-automation.md)
- [Issue #268: HAProxy HTTPS Automation](https://github.com/flyemsafe/rodhouse-datacenter/issues/268)
- [Example Playbook](../examples/configure-haproxy-for-app.yml)
- [rhdc-inventory Deployment](../../../../../rodhouse-datacenter/playbooks/pfsense-deploy-haproxy-rhdc-inventory.yml)

## Testing

```bash
# Test with example playbook
cd /home/radmin/Workspaces/datacenter/rhdc-ansible/ansible-collection-network
ansible-playbook -i inventory/hosts roles/pfsense/examples/configure-haproxy-for-app.yml

# Deploy for rhdc-inventory
cd /home/radmin/Workspaces/datacenter/rodhouse-datacenter
ansible-playbook -i inventory/hosts playbooks/pfsense-deploy-haproxy-rhdc-inventory.yml

# Verify HAProxy status
curl -k https://inventory.lab.rodhouse.net/api/health
```

---

**Created**: January 19, 2025
**Issue**: #268
**Status**: Ready for testing
