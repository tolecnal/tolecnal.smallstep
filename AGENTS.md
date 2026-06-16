# Additional AI Agent Rules

The following rules exist because automated code generation frequently introduces patterns that reduce maintainability, idempotence, or reliability.

## Do Not Ignore Errors

Do not use:

```yaml
ignore_errors: true

unless explicitly requested by a human.

Preferred approaches:

- Fix the underlying failure condition.
- Use `failed_when`.
- Use `block` and `rescue`.
- Validate prerequisites before execution.

---

## Ansible Strict Behavior and Deprecation Warnings

To maintain compatibility with future Ansible releases and strict execution modes:

1. **Strict Conditionals**: Never use raw string conditionals when evaluating boolean flags passed from the CLI (`-e flag=true`). Always cast boolean variables using `| bool`. Failure to do so will result in `ALLOW_BROKEN_CONDITIONALS` errors in modern Ansible versions.
   - Avoid: `when: my_flag or not my_file.stat.exists`
   - Preferred: `when: my_flag | bool or not my_file.stat.exists`

2. **Fact Injection Deprecation**: Avoid using injected top-level facts (e.g., `ansible_fqdn`, `ansible_os_family`). These trigger `INJECT_FACTS_AS_VARS` deprecation warnings as they will be removed in Ansible 2.24.
   - Avoid: `my_var: "{{ ansible_fqdn }}"`
   - Preferred: `my_var: "{{ ansible_facts['fqdn'] }}"`

---

## Avoid Shell Commands

Never use:

```yaml
ansible.builtin.shell:

when a suitable Ansible module exists.

Preferred order:

1. Dedicated Ansible module
2. `ansible.builtin.command`
3. `ansible.builtin.shell`

Any use of `shell` must include a justification comment.

Example:

```yaml
# No suitable module exists for this vendor utility.
- name: Run vendor configuration tool
  ansible.builtin.shell:
    cmd: /opt/vendor/bin/configure.sh

---

## Prefer Native Module Functionality

When a module exposes a parameter for a configuration setting, use that parameter instead of modifying configuration files with generic text-processing modules.

Preferred:

```yaml
- name: Configure service
  community.general.ini_file:
    path: /etc/example.conf
    section: server
    option: port
    value: "8080"

Or even better:

```yaml
- name: Create user
  ansible.builtin.user:
    name: appuser
    shell: /bin/bash
    create_home: true

Avoid:

```yaml
- name: Modify configuration using regex
  ansible.builtin.lineinfile:
    path: /etc/example.conf
    regexp: "^port="
    line: "port=8080"

when a dedicated module or module parameter already exists.

Avoid:

```yaml
- name: Change user shell
  ansible.builtin.command:
    cmd: usermod -s /bin/bash appuser

when:

```yaml
ansible.builtin.user:

supports the required functionality directly.

### Preferred Implementation Order

1. Dedicated Ansible module
2. Dedicated module parameter
3. Template
4. `community.general.ini_file`
5. `ansible.builtin.lineinfile`
6. `ansible.builtin.replace`
7. `ansible.builtin.command`
8. `ansible.builtin.shell`

Any implementation that skips a higher-preference option should include a comment explaining why.

---

## Document Task State Overrides

Any task using:

```yaml
changed_when:
failed_when:

must include a comment explaining why the override is necessary.

Example:

```yaml
# Command always returns rc=0 and provides status through stdout.
- name: Check cluster status
  ansible.builtin.command:
    cmd: clusterctl status
  register: cluster_status
  changed_when: false

---

## Variable Documentation

Whenever a new variable is introduced:

1. Add it to `defaults/main.yml` if user configurable.
2. Document it in `README.md`.
3. Provide a sensible default whenever possible.

Example:

```yaml
myrole_listen_port: 8080

Variables must not be introduced without documentation.

---

## Assertions for Required Variables

Required variables must be validated early.

Example:

```yaml
- name: Validate required variables
  ansible.builtin.assert:
    that:
      - myrole_api_key | length > 0
    fail_msg: myrole_api_key must be defined

Do not rely on task failures later in execution to identify missing configuration.

---

## Explicit State Management

Prefer declarative state management.

Preferred:

```yaml
ansible.builtin.package:
  name: nginx
  state: present

Avoid:

```yaml
ansible.builtin.command:
  cmd: apt-get install nginx

Use:

```yaml
state: present
state: absent
state: started
state: stopped
state: enabled

where supported by modules.

---

## Handler Usage Requirements

Service restarts and reloads must use handlers.

Do not restart services directly from normal tasks when handlers are appropriate.

Preferred:

```yaml
- name: Deploy configuration
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/app/app.conf
  notify:
    - Restart application

---

## Role Variable Namespacing

All role-specific variables must be prefixed with the role name.

Example:

```yaml
myrole_enabled: true
myrole_packages: []
myrole_config_file: /etc/myrole/config.yml

Avoid generic names such as:

```yaml
enabled:
packages:
config_file:

---

## Avoid Redundant Fact Gathering

Do not gather or reference facts that are not required.

When fact usage is limited, prefer targeted logic rather than broad platform-specific branching.

---

## No Version-Specific Logic Without Justification

Avoid adding operating system, distribution, or version-specific conditions unless required.

When defining OS-specific things, you must always include the proper conditionals (`when: ansible_facts...`), even inside handlers. Never define OS-specific resources or commands unconditionally.

Any platform-specific implementation should include a comment explaining the requirement.

Example:

```yaml
# Debian 12 requires package xyz for systemd integration.
when:
  - ansible_facts.distribution == 'Debian'
  - ansible_facts.distribution_major_version == '12'

---

## Maintain Idempotence

Tasks must converge cleanly.

The following command should report zero changes on the second run:

```bash
ansible-playbook site.yml

Artificial suppression of changes is not an acceptable substitute for idempotent task design.

---

## Molecule Expectations

When Molecule is present:

- Update scenarios when functionality changes.
- Ensure converge succeeds.
- Ensure idempotence succeeds.
- Ensure verify succeeds if implemented.

Expected workflow:

```bash
ansible-lint
molecule test

---

## Security Requirements

Always use:

```yaml
no_log: true

for:

- Passwords
- API keys
- Tokens
- Private keys
- Secrets of any kind

Never hardcode credentials.

Never commit example credentials that could be mistaken for real values.

---

## Repository Modification Rules

Do not:

- Reorganize the repository unless requested.
- Rename variables without updating documentation.
- Introduce new dependencies without justification.
- Add collections or roles unnecessarily.

Keep changes focused on the requested task.

---

## Readability and Line Length

- Ensure code is easily readable by splitting long strings or lists across multiple lines.
- Avoid using `# noqa` to ignore line length warnings from `ansible-lint`. Instead, refactor the code (e.g., using dictionaries, YAML anchors, or line continuations) to fit within standard line length limits (typically 160 characters).

---

## Ansible Lint Expectations

Every time a change is made, `ansible-lint` must be run and passed.

Run:

```bash
ansible-lint .

Ensure all warnings and errors are resolved.

---

## Role Domain and Requirements

When modifying the `tolecnal.smallstep` role, the following foundational design choices and functional requirements must be preserved:

**Core Requirements:**

1. **CA Initialization:** The role must support setting up the CA, strictly controlled by a unique host variable (`smallstep_ca_init`).
2. **CA Re-initialization:** The role must support safely nuking and re-initializing the CA (`smallstep_ca_reinit`).
3. **Certificate Issuance:** For all client hosts, certificates must be issued dynamically if a specific toggle variable (`smallstep_issue_cert`) is set.
4. **SAN Support:** Issued certificates must include SANs for the `ansible_fqdn`, primary IP, and any custom/wildcard SANs defined by the user.
5. **Custom SANs Variable:** Additional SANs must be configurable via a dedicated list variable (`smallstep_additional_sans`).
6. **Provisioners:** The role must deploy and support the **JWK** provisioner by default, but must fully support adding an **ACME** provisioner.
7. **Best Practices:** All deployments must follow industry best practices for PKI and Ansible.
8. **Storage Path:** Certificates and keys must be securely deployed to `/etc/ssl/smallstep/` using the format `<hostname>.<crt|key>`.
9. **Lifecycle Management:** The role must explicitly support tasks for revoking a certificate (`smallstep_revoke_cert`) and manual renewal (`smallstep_renew_cert`).

**Agreed Improvements & Features:**

1. **Certificate Lifetime:** The default certificate lifetime is set to **180 days** (rather than Smallstep's default 24 hours).
2. **Root CA Trust Distribution:** The role must automatically fetch the Root CA certificate and install it into the system-wide trust store of all clients (handling Debian and RHEL paths appropriately).
3. **Service Restart Hooks:** Client nodes must support reloading/restarting specific services automatically (e.g., Nginx, PostgreSQL) whenever their certificate is renewed.
4. **Automated Renewal:** Certificates must be kept alive hands-off through an automated Systemd timer.
5. **Secure Binary Management:** `step` and `step-ca` binaries should be reliably fetched and installed via official release URLs.
6. **Strict Secret Management:** CA passwords and provisioner passwords must be passed via variables so they can be seamlessly vaulted (e.g., Ansible Vault), with `no_log: true` strictly enforced on tasks handling them.

---

## Release Management and Changelog

This project strictly maintains a `CHANGELOG.md` following the [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) format.

- All new changes, fixes, and features must be documented under the `## [Unreleased]` section.
- When preparing for a new release, the `Unreleased` section is renamed to the new version tag (e.g., `## [1.0.0] - 2026-06-16`).

**Release Due Diligence:**
Before making a new release, the following due diligence must be performed to ensure the codebase is up to snuff:

1. Run and pass `ansible-lint .` with zero errors or warnings.
2. Perform a comprehensive security review to ensure no secrets are leaked and all endpoints/binaries are properly secured and validated.
3. Verify Molecule tests (if present) run successfully.
4. Update the `CHANGELOG.md` to reflect the new release version and date.

**Important Git Rules:**

- **Agents must NEVER perform `git tag` or push to remotes.** Release tagging and pushing are strictly reserved for manual execution by humans.



## Markdown File Requirements

All Markdown files (`.md`) must follow standard markdown specifications.

- Ensure `markdownlint` passes locally with no errors.
- Adhere to formatting standards (e.g. headers surrounded by blank lines, proper fencing, proper heading increments).
