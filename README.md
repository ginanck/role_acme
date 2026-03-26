# role_acme

Ansible role for automated ACME certificate issuance, deployment, and renewal using certbot.

## Features

- **Multiple ACME providers**: Let's Encrypt (default), ZeroSSL, Buypass, or any custom ACME-compatible CA
- **Staging/production toggle**: Test with staging endpoints to avoid rate limits
- **Challenge types**: HTTP-01 (webroot), HTTP-01 (standalone), DNS-01 (with pluggable DNS providers)
- **Multi-domain certificates**: SAN support for multiple domains per certificate
- **Wildcard certificates**: Via DNS-01 challenge
- **RSA and ECDSA keys**: Configurable key type and size (RSA 2048/4096, ECDSA P-256/P-384)
- **Combined PEM output**: Generate single bundled cert+key files for HAProxy-style consumers
- **Automated renewal**: Via systemd timer (default) or cron job
- **Certificate validity checking**: Scheduled job checks actual cert expiry and renews when needed
- **Pre/post/deploy hooks**: Run custom scripts before, after, and on successful renewal
- **Certificate backup**: Automatic backup before renewal with configurable retention
- **Monitoring script**: Nagios/Zabbix/Prometheus-compatible expiry check with JSON output
- **Configurable paths and permissions**: Full control over where certs are deployed and who can read them

## Supported Platforms

- AlmaLinux 9, 10
- RockyLinux 10
- Debian 11, 12
- Ubuntu 22.04, 24.04

## Requirements

- `role_base` must run before this role
- For HTTP-01 challenge: a web server serving `/.well-known/acme-challenge/` from the webroot
- For DNS-01 challenge: DNS provider API credentials
- For standalone challenge: port 80 must be available

## Role Variables

### ACME Provider

| Variable | Default | Description |
|---|---|---|
| `acme_provider` | `letsencrypt` | ACME provider: `letsencrypt`, `zerossl`, `buypass`, `custom` |
| `acme_email` | `""` | Contact email for ACME account (required) |
| `acme_staging` | `false` | Use staging endpoints for testing |
| `acme_custom_server_url` | `""` | Custom ACME server URL (when provider is `custom`) |
| `acme_agree_tos` | `true` | Accept provider terms of service |

### Certificate Definitions

```yaml
acme_certificates:
  - name: example.com
    domains:
      - example.com
      - www.example.com
    challenge: http-01          # http-01, dns-01, standalone
    wildcard: false             # requires dns-01
    key_type: ecdsa             # rsa, ecdsa
    key_size: 256               # RSA: 2048/4096, ECDSA: 256/384
    deploy_services:            # services to reload after issuance
      - haproxy
    combine_pem: true           # generate combined PEM for HAProxy
    combine_pem_path: ""        # custom path (default: acme_combined_pem_dir/<name>.pem)
    enabled: true
```

### Challenge Configuration

| Variable | Default | Description |
|---|---|---|
| `acme_http01_webroot` | `/var/www/acme-challenge` | Webroot for HTTP-01 challenges |
| `acme_http01_create_webroot` | `true` | Create the webroot directory |
| `acme_standalone_http_port` | `80` | Port for standalone HTTP server |
| `acme_dns01_provider` | `""` | DNS plugin: `cloudflare`, `route53`, `digitalocean`, etc. |
| `acme_dns01_credentials_file` | `""` | Path to DNS credentials on target |
| `acme_dns01_credentials` | `{}` | Credentials content (use vault) |
| `acme_dns01_propagation_seconds` | `60` | DNS propagation wait time |

### Key Settings

| Variable | Default | Description |
|---|---|---|
| `acme_default_key_type` | `ecdsa` | Default key type |
| `acme_default_key_size` | `256` | Default key size |
| `acme_preferred_chain` | `""` | Preferred certificate chain |

### Paths

| Variable | Default | Description |
|---|---|---|
| `acme_config_dir` | `/etc/letsencrypt` | Certbot config directory |
| `acme_cert_dir` | `/etc/letsencrypt/live` | Live certificate directory |
| `acme_combined_pem_dir` | `/etc/ssl/acme` | Combined PEM output directory |
| `acme_log_dir` | `/var/log/letsencrypt` | Log directory |

### Certificate Deployment

| Variable | Default | Description |
|---|---|---|
| `acme_cert_owner` | `root` | Certificate file owner |
| `acme_cert_group` | `root` | Certificate file group |
| `acme_cert_mode` | `0640` | Certificate file mode |
| `acme_key_mode` | `0600` | Private key file mode |
| `acme_combined_pem_mode` | `0640` | Combined PEM file mode |

### Automated Renewal

| Variable | Default | Description |
|---|---|---|
| `acme_renewal_enabled` | `true` | Enable automated renewal |
| `acme_renewal_method` | `systemd` | Renewal method: `systemd` or `cron` |
| `acme_renewal_days_before_expiry` | `30` | Days before expiry to trigger renewal |
| `acme_renewal_systemd_calendar` | `*-*-* 00,12:00:00` | Systemd timer schedule |
| `acme_renewal_systemd_random_delay` | `43200` | Random delay in seconds |
| `acme_renewal_cron_hour` | `0,12` | Cron hour (when using cron) |

### Renewal Hooks

| Variable | Default | Description |
|---|---|---|
| `acme_renewal_pre_hook` | `""` | Shell command to run before renewal |
| `acme_renewal_post_hook` | `""` | Shell command to run after renewal |
| `acme_renewal_deploy_hooks` | `[]` | List of deploy hook scripts |

### Backup

| Variable | Default | Description |
|---|---|---|
| `acme_backup_enabled` | `true` | Back up certs before renewal |
| `acme_backup_dir` | `/var/backups/acme` | Backup directory |
| `acme_backup_retain` | `5` | Number of backups to keep |

### Monitoring

| Variable | Default | Description |
|---|---|---|
| `acme_monitoring_enabled` | `true` | Deploy monitoring script |
| `acme_monitoring_script_path` | `/usr/local/bin/acme-cert-check` | Script path |
| `acme_monitoring_warning_days` | `14` | Warning threshold |
| `acme_monitoring_critical_days` | `7` | Critical threshold |
| `acme_monitoring_status_file` | `/var/run/acme-renewal-status` | Status file path |

### Safety

| Variable | Default | Description |
|---|---|---|
| `acme_dry_run` | `false` | Test without issuing certificates |
| `acme_force_renewal` | `false` | Force renewal even if not near expiry |

## Example Playbooks

### Basic HTTP-01 with HAProxy

```yaml
- hosts: loadbalancers
  become: true
  vars:
    acme_email: admin@example.com
    acme_certificates:
      - name: example.com
        domains:
          - example.com
          - www.example.com
        challenge: http-01
        combine_pem: true
        deploy_services:
          - haproxy
  roles:
    - role_base
    - role_acme
    - role_haproxy
```

### Wildcard Certificate with DNS-01 (Cloudflare)

```yaml
- hosts: loadbalancers
  become: true
  vars:
    acme_email: admin@example.com
    acme_dns01_provider: cloudflare
    acme_dns01_credentials_file: /etc/letsencrypt/cloudflare.ini
    acme_dns01_credentials:
      dns_cloudflare_api_token: "{{ vault_cloudflare_api_token }}"
    acme_certificates:
      - name: example.com
        domains:
          - example.com
          - "*.example.com"
        challenge: dns-01
        wildcard: true
        key_type: ecdsa
        key_size: 384
        combine_pem: true
        combine_pem_path: /etc/haproxy/certs/example.com.pem
        deploy_services:
          - haproxy
  roles:
    - role_base
    - role_acme
```

### Multiple Certificates with Different Methods

```yaml
- hosts: loadbalancers
  become: true
  vars:
    acme_email: admin@example.com
    acme_dns01_provider: cloudflare
    acme_dns01_credentials_file: /etc/letsencrypt/cloudflare.ini
    acme_dns01_credentials:
      dns_cloudflare_api_token: "{{ vault_cloudflare_api_token }}"
    acme_certificates:
      - name: public.example.com
        domains:
          - public.example.com
        challenge: http-01
        combine_pem: true
        deploy_services:
          - nginx
      - name: internal.example.com
        domains:
          - internal.example.com
          - "*.internal.example.com"
        challenge: dns-01
        wildcard: true
        combine_pem: true
        deploy_services:
          - haproxy
  roles:
    - role_base
    - role_acme
```

### Staging Mode for Testing

```yaml
- hosts: loadbalancers
  become: true
  vars:
    acme_email: admin@example.com
    acme_staging: true
    acme_certificates:
      - name: example.com
        domains:
          - example.com
        challenge: standalone
  roles:
    - role_base
    - role_acme
```

## Monitoring Script Usage

The deployed monitoring script supports multiple output formats:

```bash
# Verbose check of all certificates
acme-cert-check

# Quiet mode (only warnings/criticals)
acme-cert-check --quiet

# JSON output for automation
acme-cert-check --json

# Nagios-compatible output
acme-cert-check --nagios
```

Exit codes: `0` = OK, `1` = WARNING, `2` = CRITICAL, `3` = UNKNOWN

## Dependencies

- `role_base`

## License

GPL-2.0-or-later

## Author

gkorkmaz
