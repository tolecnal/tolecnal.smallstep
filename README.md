# tolecnal.smallstep

Ansible role to setup an internal Smallstep CA and issue certificates.

## Features

- CA Initialization & Configuration
- Supports JWK and ACME provisioners natively
- Cross-platform step & step-ca binary installation
- Automatic distribution of Root CA to system trust store
- Automatic certificate issuance
- Customizable SANs (Subject Alternative Names) including wildcards
- Automatic background certificate renewal via systemd timers
- Service hooks upon certificate renewal (e.g. reload Nginx/Postgres)
- Certificate revocation capabilities

## Role Variables Matrix

To help clarify which variables apply to which mode, here is a breakdown of the requirements for each operational mode:

### Root CA (Offline)

Must be used to set up the top-level offline CA.

- **Required**: `smallstep_ca_init: true`, `smallstep_ca_type: "root"`, `smallstep_ca_password`
- **Optional**: `smallstep_ca_name`, `smallstep_ca_cert_lifetime`

### Intermediate CA (Online)

Must be used to set up the online CA daemon that clients connect to.

- **Required**: `smallstep_ca_init: true`, `smallstep_ca_type: "intermediate"`, `smallstep_root_ca_host` (must match the Ansible inventory name of the Root CA), `smallstep_ca_password`, `smallstep_ca_dns`
- **Optional**: `smallstep_ca_name`, `smallstep_ca_url`, `smallstep_ca_address`, `smallstep_provisioners`, `smallstep_ca_cert_lifetime`

### Standalone CA (Root + Intermediate)

The default mode. Sets up the Root and Intermediate CAs natively on the same machine.

- **Required**: `smallstep_ca_init: true`, `smallstep_ca_password`
- **Optional**: `smallstep_ca_type: "standalone"`, `smallstep_ca_name`, `smallstep_ca_dns`, `smallstep_ca_url`, `smallstep_ca_address`, `smallstep_provisioners`, `smallstep_ca_cert_lifetime`

### Client (Certificate Request)

Used to issue, renew, or revoke a certificate for a client host.

- **Required**: `smallstep_issue_cert: true`, `smallstep_ca_url`, `smallstep_provisioner_password` (if using JWK)
- **Optional**: `smallstep_ca_fingerprint` (auto-fetched if omitted), `smallstep_cert_name`, `smallstep_additional_sans`, `smallstep_provisioner_name` (defaults to `ansible-jwk`), `smallstep_reload_services`

| Variable | Description | Default |
| -------- | ----------- | ------- |
| `smallstep_ca_init` | Set to true to initialize this host as the CA. | `false` |
| `smallstep_ca_type` | Operational mode for the CA. Options: `standalone`, `root`, `intermediate`. | `standalone` |
| `smallstep_root_ca_host`| Ansible inventory hostname of the Root CA. Required if `smallstep_ca_type` is `intermediate`. | `""` |
| `smallstep_ca_reinit` | Set to true to force a clean re-initialization of the CA. | `false` |
| `smallstep_issue_cert` | Set to true to issue a certificate and setup auto-renewal for this host. | `false` |
| `smallstep_sign_csr` | Set to true to securely sign an offline CSR file provided in `smallstep_csr_path`. | `false` |
| `smallstep_issue_offline` | Set to true to issue a certificate purely as static files without setting up timers. | `false` |
| `smallstep_revoke_cert`| Set to true to revoke the host's certificate. | `false` |
| `smallstep_renew_cert` | Set to true to manually trigger an immediate renewal. | `false` |
| `smallstep_cli_version` | Version of the `step` CLI to install. | `0.26.1` |
| `smallstep_ca_version` | Version of the `step-ca` binary to install. | `0.26.0` |
| `smallstep_ca_name` | Name of the internal CA. | `Internal CA` |
| `smallstep_ca_dns` | DNS name of the CA server. | `{{ ansible_facts['fqdn'] }}` |
| `smallstep_ca_additional_sans` | Additional SANs (DNS names) for the CA certificates. | `[]` |
| `smallstep_ca_crl_enabled` | Enable the native CRL generation endpoint at `/1.0/crl`. | `false` |
| `smallstep_ca_crl_cache_duration` | The duration for which the CRL remains valid. | `24h` |
| `smallstep_ca_address` | Bind address and port for the CA server. | `:443` |
| `smallstep_ca_url` | Full HTTPS URL of the CA. | `https://{{ smallstep_ca_dns }}` |
| `smallstep_ca_password`| Required vault variable containing the CA password. Deployed with 0600 permissions. | `""` |
| `smallstep_ca_fingerprint` | Root CA fingerprint. Required for clients to securely bootstrap trust. | `""` |
| `smallstep_ca_cert_lifetime` | Default & max certificate lifetime. | `4320h` (180 days) |
| `smallstep_permitted_dns_domains`| List of DNS domains to permit via X.509 Name Constraints. | `[]` |
| `smallstep_permitted_ip_ranges`| List of IP CIDRs to permit via X.509 Name Constraints. | `["0.0.0.0/0", "::/0"]` |
| `smallstep_provisioners`| List of provisioners to add to the CA. | `ansible-jwk`, `acme` |
| `smallstep_cert_name` | The common name of the certificate to issue. | `{{ ansible_facts['fqdn'] }}` |
| `smallstep_cert_path` | Where to store the certificate on disk. | `/etc/ssl/smallstep` |
| `smallstep_additional_sans`| List of additional SANs to add to the cert. | `[]` |
| `smallstep_provisioner_name`| Which provisioner to use to issue the cert. | `ansible-jwk` |
| `smallstep_provisioner_password`| Vaulted variable with the JWK password. | `""` |
| `smallstep_reload_services`| List of systemd services to reload on renewal. | `[]` |
| `smallstep_csr_path` | Absolute path to the offline CSR file to be signed when using `smallstep_sign_csr`. | `""` |

```yaml
- hosts: ca_servers
  become: true
  roles:
  - tolecnal.smallstep
  vars:
    smallstep_ca_init: true
    smallstep_ca_password: "{{ vault_smallstep_ca_password }}"
    smallstep_provisioner_password: "{{ vault_smallstep_jwk_password }}"
    smallstep_provisioners:
    - name: "ansible-jwk"
        type: "JWK"
    - name: "acme"
        type: "ACME"

```

### For Clients

**Note**: The role will automatically fetch the Root CA fingerprint directly from the CA server via SSH (using Ansible's `delegate_to`) if you omit the `smallstep_ca_fingerprint` variable. It defaults to using the hostname found in your `smallstep_ca_url`. If your CA server has a different hostname in your Ansible inventory, you can specify it using `smallstep_ca_delegate_host`.

```yaml
- hosts: web_servers
  become: true
  gather_facts: true
  roles:
    - tolecnal.smallstep
  vars:
    smallstep_issue_cert: true
    smallstep_ca_url: "https://ca.example.internal"
    # smallstep_ca_delegate_host: "ca_servers_inventory_name" # Optional if different from URL
    smallstep_provisioner_password: "{{ vault_smallstep_jwk_password }}"
    smallstep_additional_sans:
      - "www.example.internal"
      - "*.example.internal"
    smallstep_reload_services:
      - nginx
```

### Multi-Host CA Architecture (Enterprise PKI)

For increased security, you can isolate the Root CA onto its own offline server and run the `step-ca` daemon on a separate Intermediate server. The role automatically securely fetches and signs the CSR across both hosts using Ansible orchestration.

```yaml
- name: Deploy Offline Root CA
  hosts: root_ca
  become: true
  roles:
    - tolecnal.smallstep
  vars:
    smallstep_ca_init: true
    smallstep_ca_type: "root"
    smallstep_ca_password: "{{ vault_ca_password }}"

- name: Deploy Online Intermediate CA
  hosts: intermediate_ca
  become: true
  roles:
    - tolecnal.smallstep
  vars:
    smallstep_ca_init: true
    smallstep_ca_type: "intermediate"
    smallstep_root_ca_host: "root_ca" # Name of the Root CA host in your Ansible inventory
    smallstep_ca_dns: "ca.example.internal"
    smallstep_ca_url: "https://ca.example.internal"
    smallstep_ca_password: "{{ vault_ca_password }}"
    smallstep_provisioners:
      - name: "ansible-jwk"
        type: "JWK"
        password: "{{ vault_provisioner_password }}"
      - name: "acme"
        type: "ACME"
```

*(Note: You must converge the Root CA and Intermediate CA in the same playbook or ensure the Root CA runs first).*

## Security Notes

1. **Passwords**: `smallstep_ca_password` and `smallstep_provisioner_password` must be vaulted. The CA password is kept permanently on disk at `/etc/step-ca/password.txt` with strict `0600` permissions (owned by the `step` user) to allow the `step-ca` systemd service to automatically restart gracefully.
2. **Bootstrapping**: Clients mandate the `smallstep_ca_fingerprint` variable to securely bootstrap trust and download the Root CA. Without this, the role will fail the client installation to prevent MITM attacks.
3. **No Log**: All tasks handling secrets enforce `no_log: true` to keep your Ansible logs pristine.

## Active Revocation (CRL)

While Smallstep strongly advocates for short-lived certificates over active revocation checks, `step-ca` does natively support generating a **Certificate Revocation List (CRL)**.

You can enable this feature at any time by setting `smallstep_ca_crl_enabled: true` and re-running the CA playbook.

**Important details regarding CRL enablement:**

- **No CA Re-init Required**: Enabling this dynamically patches your `ca.json` and restarts the service. It does not rebuild your CA or destroy your keys.
- **Existing Certificates Remain Valid**: Issued certificates are untouched and remain perfectly valid.
- **No Client Renewals Required**: To maintain compatibility and robustness, this role relies on the default `step-ca` templates, which **do not** hardcode the CRL Distribution Point (CDP) into the certificate payloads. Instead, relying services (like Nginx, HAProxy, or API Gateways) should be explicitly configured to fetch and enforce the CRL directly from `https://<ca_url>/1.0/crl`.

## Trusting the CA on Non-Ansible Clients (macOS / Windows)

For client devices that aren't managed by Ansible (like laptops), there are two primary ways to securely download and trust the Root CA.

### Option 1: Using the `step` CLI (Recommended)

If the user has the `step` CLI installed (e.g., via `brew install step`), the CLI has native functionality to download the certificate and automatically inject it into the macOS System Keychain or Windows Certificate Store!

```bash
# 1. Download the Root CA securely
step ca root root_ca.crt --ca-url="https://ca.example.internal" --fingerprint="YOUR_ROOT_FINGERPRINT"

# 2. Install it into the system trust store (will prompt for admin credentials)
step certificate install root_ca.crt
```

### Option 2: Direct Download

If the user doesn't have the CLI installed, the CA serves its root certificates natively over HTTPS at the `/roots.pem` endpoint.

```bash
curl -kO https://ca.example.internal/roots.pem
```

*(The `-k` is required because the machine doesn't trust the server yet).*

Once downloaded:

1. **macOS**: Double-click `roots.pem` to open Keychain Access. Find the CA, double-click it, expand the "Trust" section, and change "When using this certificate" to **Always Trust**.
2. **Windows**: Double-click `roots.pem` -> Install Certificate -> Place all certificates in the following store -> Browse -> **Trusted Root Certification Authorities**.

## Offline Certificate Workflows (Appliances & Firewalls)

For devices like network switches, firewalls, or ILO/iDRAC interfaces that do not support ACME or installing the `step` CLI natively, you can issue certificates "offline" on your Ansible Controller (or any proxy machine) and manually upload them to the appliance.

### Method 1: CSR Signing (Most Secure)

If your appliance can generate a Private Key and a CSR:

1. Generate the CSR on the appliance and save it locally (e.g. `/tmp/appliance.csr`).
2. Run the `sign_csr.yml` playbook locally:

```yaml
- name: Sign Offline CSR
  hosts: localhost
  become: true
  roles:
    - tolecnal.smallstep
  vars:
    smallstep_sign_csr: true
    smallstep_ca_url: "https://ca.example.internal"
    smallstep_csr_path: "/tmp/appliance.csr"
    smallstep_cert_name: "appliance.example.internal"
    smallstep_cert_path: "/tmp/certs"
```

The role will sign the CSR and drop `appliance.example.internal.crt` into `/tmp/certs`.

### Method 2: Direct Offline Generation

If your appliance expects you to upload both the Private Key and Certificate:

```yaml
- name: Issue Offline Certificate
  hosts: localhost
  become: true
  roles:
    - tolecnal.smallstep
  vars:
    smallstep_issue_offline: true
    smallstep_ca_url: "https://ca.example.internal"
    smallstep_cert_name: "switch.example.internal"
    smallstep_cert_path: "/tmp/certs"
    smallstep_additional_sans:
      - "192.168.1.50"
```

The role will securely generate both the `.crt` and `.key` in `/tmp/certs` without configuring any systemd timers or renewal services.

## ACME Provisioning Guide

By default, the role configures two provisioners on the CA:

1. `ansible-jwk`: A standard JWK provisioner used for automated certificate issuance via Ansible using a vaulted password.
2. `acme`: A standard ACME protocol provisioner that mimics Let's Encrypt, allowing third-party tools to fetch certificates dynamically.

### Using ACME via Third-Party Tools (e.g. Certbot, Traefik, Caddy)

Any standard ACME client can request certificates from your internal CA, provided it trusts your Root CA.
The ACME Directory URL for your internal CA is typically:

```text
https://ca.example.internal/acme/acme/directory
```

**Example using Certbot:**

```bash
# Provide the ACME directory URL and instruct Certbot to trust your downloaded Root CA
REQUESTS_CA_BUNDLE=/etc/ssl/certs/smallstep_root_ca.crt \
certbot certonly -d my-service.example.internal \
  --server https://ca.example.internal/acme/acme/directory \
  --standalone
```

**Example using Traefik:**

```yaml
certificatesResolvers:
  smallstep:
    acme:
      email: admin@example.internal
      storage: acme.json
      caServer: "https://ca.example.internal/acme/acme/directory"
      tlsChallenge: {}
```

*(Note: You must ensure that the container running Traefik mounts or trusts the Root CA certificate).*

### Using ACME inside this Ansible Role

While JWK is the recommended method for issuing certificates natively via Ansible (as it does not require opening ports for challenges), you can instruct the role to use ACME for a specific host by overriding the `smallstep_provisioner_name` variable in your client playbook:

```yaml
- hosts: web_servers
  become: true
  roles:
    - tolecnal.smallstep
  vars:
    smallstep_issue_cert: true
    smallstep_ca_url: "https://ca.example.internal"
    smallstep_provisioner_name: "acme"
```

*Note: Using ACME directly via the `step ca certificate` command (which this role runs underneath) requires port `80` or `443` to be temporarily available on the client host to pass the ACME HTTP-01 or ALPN challenge.*
