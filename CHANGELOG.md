# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-06-16

### Added

- Initial release of the `tolecnal.smallstep` Ansible role.
- Support for multi-host Enterprise PKI architectures (offline Root CA signing online Intermediate CAs via Ansible delegation).
- Support for strict cryptographic X.509 Name Constraints (domain and IP whitelisting) injected into the Intermediate CA.
- Support for automated initialization and secure re-initialization of a Smallstep Certificate Authority.
- Support for issuing, renewing, and revoking client certificates using JWK and ACME provisioners.
- Automated client side Root CA trust store distribution for Debian and RedHat families.
- Systemd timer implementation for hands-off automatic certificate renewal.
- Dynamic SAN generation combining FQDN, primary IP address, and user-defined custom SANs.
- Extensive guardrails against deprecation warnings and strict boolean conditional failures.
- Molecule testing suite with Docker backend to validate CA and client deployments.
- Comprehensive `README.md` and `AGENTS.md` to document usage and design constraints.
