# FreeIPA: Beginner to Expert — RHEL 10

> A complete, hands-on course covering FreeIPA installation, identity management,
> Kerberos authentication, DNS/DNSSEC, RBAC, certificate management (Dogtag + Certmonger),
> AD trust, replication, security hardening, and operational recipes.

**Platform:** RHEL 10 only  
**FreeIPA version:** 4.12.x  
**Dogtag PKI:** 11.x  
**BIND:** 9.18.x  
**Python:** 3.12  

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![FreeIPA](https://img.shields.io/badge/FreeIPA-v4.12-blue)](https://www.freeipa.org)

---

## Lab Environment Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| vCPU | 2 | 4 |
| RAM | 4 GB | 8 GB |
| Disk | 20 GB | 40 GB |
| OS | RHEL 10 | RHEL 10 |
| Network | Static IP | Static IP |

### DNS Pre-requisites (before install)
- Fully-qualified hostname set: `hostnamectl set-hostname ipa.example.com`
- Forward DNS resolves the server's FQDN to its IP
- Reverse DNS resolves the server's IP to its FQDN
- `/etc/hosts` contains `<IP>  ipa.example.com  ipa`
- NTP/Chrony synchronised (clock skew < 5 min)

### Lab Environment Naming Convention

All examples in this course use the following canonical hostnames and IPs. Adjust to match your actual environment.

| Role | Hostname | IP Address | Notes |
|------|----------|------------|-------|
| IPA Server 1 (primary) | `ipa1.example.com` | 192.168.1.10 | First IPA master |
| IPA Server 2 (replica) | `ipa2.example.com` | 192.168.1.11 | Replica / CA replica |
| Optional VIP / CNAME | `ipa.example.com` | → `ipa1.example.com` | Stable alias used in single-server install examples |
| IPA Client (generic) | `client01.example.com` | 192.168.1.20 | Linux host enrolled in IPA |
| Web server (example) | `web01.example.com` | 192.168.1.21 | App host |
| Database (example) | `db01.example.com` | 192.168.1.22 | DB host |
| AD Domain Controller | `dc1.ad.example.com` | 192.168.10.10 | AD DC for trust (Module 11) |
| AD DC 2 (secondary) | `dc2.ad.example.com` | 192.168.10.11 | Optional second AD DC |

| Setting | Value |
|---------|-------|
| IPA DNS domain | `example.com` |
| IPA Kerberos realm | `EXAMPLE.COM` |
| AD DNS domain | `ad.example.com` |
| AD Kerberos realm | `AD.EXAMPLE.COM` |

> 📝 **Kerberos realms are always UPPERCASE; DNS domains are always lowercase.**

### Firewall Ports (IPA Server)

| Port | Protocol | Service |
|------|----------|---------|
| 80, 443 | TCP | HTTP/HTTPS (IPA web UI + API) |
| 389, 636 | TCP | LDAP / LDAPS |
| 88, 464 | TCP+UDP | Kerberos KDC / kpasswd |
| 53 | TCP+UDP | DNS |
| 123 | UDP | NTP |
| 7389 | TCP | Dogtag CA (internal) |

---

## Course Modules

| # | File | Topic | Level |
|---|------|-------|-------|
| 00 | [00_introduction.md](00_introduction.md) | What is FreeIPA, components, use cases | Beginner |
| 01 | [01_architecture.md](01_architecture.md) | Server/replica/client, internal wiring, IPA API | Beginner |
| 02 | [02_installation.md](02_installation.md) | Prerequisites, `ipa-server-install`, first login | Beginner |
| 03 | [03_identity_users_groups.md](03_identity_users_groups.md) | Users, groups, automember, ID ranges | Beginner |
| 04 | [04_authentication_kerberos.md](04_authentication_kerberos.md) | Kerberos deep-dive, OTP, 2FA, password policy | Intermediate |
| 05 | [05_host_enrollment_sssd.md](05_host_enrollment_sssd.md) | Client install, SSSD, PAM/NSS, offline auth | Intermediate |
| 06 | [06_dns_dnssec.md](06_dns_dnssec.md) | Integrated DNS, DNSSEC, dynamic updates | Intermediate |
| 07 | [07_sudo_hbac.md](07_sudo_hbac.md) | Sudo rules, Host-Based Access Control | Intermediate |
| 08 | [08_rbac_delegation.md](08_rbac_delegation.md) | RBAC deep-dive: permissions, privileges, roles, ACIs | Advanced |
| 09 | [09_certificate_management_fundamentals.md](09_certificate_management_fundamentals.md) | Dogtag CA, profiles, `ipa cert-*`, Certmonger | Advanced |
| 10 | [10_certificate_management_advanced.md](10_certificate_management_advanced.md) | Sub-CAs, ACME, external CA, OCSP, FIPS | Advanced |
| 11 | [11_ad_trust.md](11_ad_trust.md) | AD trust types, cross-realm Kerberos, SID mapping | Advanced |
| 12 | [12_replication_topology.md](12_replication_topology.md) | Replicas, topology segments, CA replicas | Advanced |
| 13 | [13_security_hardening.md](13_security_hardening.md) | Crypto policies, TLS 1.3, SELinux, FIPS | Expert |
| 14 | [14_troubleshooting.md](14_troubleshooting.md) | Triage trees, logs, `ipa-healthcheck`, cert expiry | Expert |
| 15 | [15_upgrade_migration.md](15_upgrade_migration.md) | IPA upgrades, rolling replica upgrade, backup/restore, LDAP/NIS/AD migration | Expert |

---

## Cheatsheets & Recipes

| File | Contents |
|------|----------|
| [cheatsheets/cs_01_daily_admin.md](cheatsheets/cs_01_daily_admin.md) | Day-to-day admin commands (users, groups, hosts, services) |
| [cheatsheets/cs_02_kerberos.md](cheatsheets/cs_02_kerberos.md) | kinit variants, keytabs, klist, common errors |
| [cheatsheets/cs_03_certificates_certmonger.md](cheatsheets/cs_03_certificates_certmonger.md) | cert-request, getcert, renewal, revoke, sub-CA, ACME |
| [cheatsheets/cs_04_dns_dnssec.md](cheatsheets/cs_04_dns_dnssec.md) | dnszone/dnsrecord commands, DNSSEC, forwarders |
| [cheatsheets/cs_05_rbac_delegation.md](cheatsheets/cs_05_rbac_delegation.md) | Permissions, privileges, roles, delegation, self-service |
| [cheatsheets/cs_06_replication.md](cheatsheets/cs_06_replication.md) | Replica install, topology, re-init, status |
| [cheatsheets/cs_07_ad_trust.md](cheatsheets/cs_07_ad_trust.md) | trust-add, ID ranges, cross-realm test, diagnostics |
| [cheatsheets/cs_08_troubleshooting.md](cheatsheets/cs_08_troubleshooting.md) | Symptom → diagnosis → fix reference |
| [cheatsheets/cs_09_recipes.md](cheatsheets/cs_09_recipes.md) | 12 step-by-step operational recipes |

---

## Learning Path

```
Beginner  ──▶  00 → 01 → 02 → 03
Intermediate ──▶  04 → 05 → 06 → 07
Advanced  ──▶  08 → 09 → 10 → 11 → 12
Expert    ──▶  13 → 14 → 15
Operations ──▶  cheatsheets/ (use as reference at any level)
```

---

## Conventions Used in This Course

- `#` — command run as **root** or with `sudo`
- `$` — command run as a **regular user** or `admin`
- `(server)` — run on the **IPA server**
- `(client)` — run on an **enrolled client**
- 📝 **Note** — important clarification
- ⚠️ **Warning** — destructive or irreversible operation
- 🔒 **FIPS** — relevant when FIPS mode is enabled
- 🔁 **See also** — cross-reference to another module or cheatsheet

All `firewall-cmd` examples use `firewalld`. `iptables` is not used.  
All container examples use `podman` (rootless). Docker is not used.  
Crypto: SHA-256 minimum. SHA-1 and MD5 are never used. No `LEGACY` crypto policy.

---

## License

This project is licensed under the
Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0).

https://creativecommons.org/licenses/by-nc-sa/4.0/

---

© 2026 UncleJS — Licensed under CC BY-NC-SA 4.0
