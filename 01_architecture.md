# Module 01 — FreeIPA Architecture

> A detailed look at how FreeIPA's components connect internally, how the server,
> replica, and client roles differ, and how requests flow through the system.

## Table of Contents

- [1. The Three Roles: Server, Replica, Client](#1-the-three-roles-server-replica-client)
  - [1.1 IPA Server](#11-ipa-server)
  - [1.2 IPA Replica](#12-ipa-replica)
  - [1.3 IPA Client](#13-ipa-client)
- [2. Internal Component Wiring](#2-internal-component-wiring)
  - [2.1 How Components Share Data](#21-how-components-share-data)
  - [2.2 Service Start Order](#22-service-start-order)
- [3. The IPA API Request Flow](#3-the-ipa-api-request-flow)
  - [3.1 Authentication to the API](#31-authentication-to-the-api)
  - [3.2 Command Execution Path](#32-command-execution-path)
- [4. CA Topology](#4-ca-topology)
  - [4.1 Root CA and Sub-CAs](#41-root-ca-and-sub-cas)
  - [4.2 CA Replica vs CA-less Replica](#42-ca-replica-vs-ca-less-replica)
- [5. Directory Information Tree (DIT) Layout](#5-directory-information-tree-dit-layout)
- [6. Replication Overview](#6-replication-overview)
- [7. Lab — Inspect a Running IPA Server](#7-lab--inspect-a-running-ipa-server)

---

## 1. The Three Roles: Server, Replica, Client

```mermaid
graph TD
    subgraph "IPA Domain: example.com / EXAMPLE.COM"
        S1[IPA Server\nipa1.example.com\nKDC + CA + LDAP + DNS]
        S2[IPA Replica\nipa2.example.com\nKDC + CA + LDAP + DNS]
        S1 <-->|multi-master replication\n389-DS + Dogtag| S2
    end
    C1[Linux Client\nweb01.example.com\nSSSD] -->|LDAP queries\nKerberos auth| S1
    C2[Linux Client\ndb01.example.com\nSSSD] -->|LDAP queries\nKerberos auth| S2
    C3[Linux Client\napp01.example.com\nSSSD] -->|failover| S1
    C3 -->|failover| S2
```

### 1.1 IPA Server

The first IPA server installed is the **initial master**. It runs:

| Component | Process | Port |
|-----------|---------|------|
| 389 Directory Server | `dirsrv@EXAMPLE-COM` | 389, 636 |
| MIT Kerberos KDC | `krb5kdc` | 88 |
| Kerberos kadmin | `kadmin` | 749 |
| Dogtag CA | `pki-tomcatd@pki-tomcat` | 8080, 8443 |
| BIND DNS | `named` | 53 |
| Apache httpd (API+UI) | `httpd` | 80, 443 |
| Certmonger | `certmonger` | — |
| SSSD (local) | `sssd` | — |
| Chronyd (NTP) | `chronyd` | 123 |

All services are managed by systemd. The `ipactl` wrapper starts/stops them
in the correct dependency order.

### 1.2 IPA Replica

A replica is a **full peer** of the IPA server. It runs the same services and
participates in multi-master replication. There is no "primary" — if ipa1 fails,
ipa2 continues serving all clients without any manual intervention.

A replica can be:
- **Full replica** — KDC + CA + DNS + LDAP (recommended)
- **CA-less replica** — KDC + DNS + LDAP, no local Dogtag instance (cert requests
  proxy to a CA replica)
- **Hidden replica** — participates in replication but is excluded from DNS SRV
  records; used for backups or maintenance

### 1.3 IPA Client

A client is any RHEL 10 host that has run `ipa-client-install`. It does **not**
run IPA server components. It runs:

- `sssd` — NSS + PAM + Kerberos + sudo + HBAC
- `certmonger` — requests and renews host/service certificates
- Kerberos client libs — `kinit`, keytab support

Clients discover IPA servers via **DNS SRV records**:
```
_ldap._tcp.example.com.   SRV  0 100 389 ipa1.example.com.
_kerberos._tcp.EXAMPLE.COM. SRV 0 100 88 ipa1.example.com.
```

[↑ Back to TOC](#table-of-contents)

---

## 2. Internal Component Wiring

### 2.1 How Components Share Data

All IPA components use **389-DS as the single source of truth**. There is no
separate database per component.

```mermaid
graph TD
    A[ipa CLI / Web UI] -->|JSON-RPC over HTTPS| B[httpd\nmod_auth_gssapi\nipa-wsgi]
    B -->|python-ldap| C[389 Directory Server\nport 636 LDAPS]
    B -->|ipa-otpd| D[MIT Kerberos KDC]
    D -->|kldap plugin reads principals| C
    E[Dogtag CA\nTomcat :8443] -->|CS.cfg\nldapAuth| C
    F[BIND DNS\nbind-dyndb-ldap plugin] -->|reads/writes zones| C
    G[Certmonger] -->|ipa-submit helper| B
    H[SSSD on clients] -->|LDAP bind| C
    H -->|kinit| D
```

Key insight: **BIND, KDC, and Dogtag all read their runtime data from 389-DS**.
When you add a DNS record with `ipa dnsrecord-add`, it writes to LDAP — BIND picks
it up via the dyndb plugin within seconds, with no zone file reload needed.

### 2.2 Service Start Order

`ipactl start` starts services in this order (dependency chain):

```mermaid
graph LR
    A[dirsrv\n389-DS] --> B[krb5kdc\nKDC]
    A --> C[pki-tomcatd\nDogtag CA]
    A --> D[named\nBIND DNS]
    B --> E[httpd\nAPI + UI]
    C --> E
    D --> E
    E --> F[ipa-otpd\nOTP daemon]
    A --> G[sssd\nlocal auth]
```

> ⚠️ If `dirsrv` fails to start, everything else fails. Troubleshoot 389-DS first.

[↑ Back to TOC](#table-of-contents)

---

## 3. The IPA API Request Flow

### 3.1 Authentication to the API

Every API call requires a valid Kerberos ticket for the `admin` principal (or any
principal with sufficient RBAC permissions).

```mermaid
sequenceDiagram
    participant U as User / CLI
    participant KDC as MIT Kerberos KDC
    participant API as httpd (IPA API)
    participant LDAP as 389-DS

    U->>KDC: AS-REQ (kinit admin)
    KDC-->>U: AS-REP (TGT for admin@EXAMPLE.COM)
    U->>KDC: TGS-REQ (ticket for HTTP/ipa1.example.com)
    KDC-->>U: TGS-REP (service ticket)
    U->>API: HTTPS POST /ipa/json\n(Authorization: Negotiate <ticket>)
    API->>API: mod_auth_gssapi validates ticket
    API-->>U: 200 OK (Kerberos negotiation complete)
```

### 3.2 Command Execution Path

Once authenticated, each `ipa` command translates to a JSON-RPC call:

```mermaid
sequenceDiagram
    participant CLI as ipa user-add jdoe
    participant API as IPA API (wsgi)
    participant RBAC as RBAC / ACI check
    participant LDAP as 389-DS
    participant Hook as Plugin hooks

    CLI->>API: POST /ipa/json\n{"method":"user_add","params":[["jdoe"],{...}]}
    API->>RBAC: Does admin have permission\n"System: Add Users"?
    RBAC-->>API: Allowed
    API->>Hook: pre_callback hooks\n(validate, normalise)
    Hook-->>API: OK
    API->>LDAP: ldap_add(dn=uid=jdoe,cn=users,...)
    LDAP-->>API: Success
    API->>Hook: post_callback hooks\n(automember eval, etc.)
    Hook-->>API: OK
    API-->>CLI: {"result":{"dn":"uid=jdoe,..."},"error":null}
```

[↑ Back to TOC](#table-of-contents)

---

## 4. CA Topology

### 4.1 Root CA and Sub-CAs

FreeIPA installs a **self-signed root CA** (the IPA CA) during `ipa-server-install`.
This CA issues certificates for all IPA services and all enrolled hosts/services.

Sub-CAs can be created beneath the root CA, useful for:
- Segregating certificates by department, environment, or service type
- Applying different certificate policies per sub-CA
- Issuing certificates that appear to come from a named sub-authority

```mermaid
graph TD
    A[IPA Root CA\ncn=Certificate Authority\nSelf-signed, 20-year validity] --> B[IPA Service Certs\nHTTP, LDAP, KDC, DNS]
    A --> C[Host Certs\nall enrolled clients]
    A --> D[Sub-CA: VPN\ncn=VPN Authority]
    A --> E[Sub-CA: Users\ncn=User Certificate Authority]
    D --> D1[VPN Client Certs]
    D --> D2[VPN Server Certs]
    E --> E1[User Smart Card Certs]
    E --> E2[User Email Signing Certs]
```

### 4.2 CA Replica vs CA-less Replica

```mermaid
graph LR
    subgraph "CA Replicas (recommended)"
        R1[ipa1\nCA + KDC + LDAP]
        R2[ipa2\nCA + KDC + LDAP]
        R1 <-->|Dogtag replication\n+ 389-DS replication| R2
    end
    subgraph "CA-less Replica"
        R3[ipa3\nKDC + LDAP only]
        R3 -->|cert requests proxied to| R1
    end
    C1[clients] --> R1
    C1 --> R2
    C2[clients] --> R3
```

> 📝 Always have at least **two CA replicas**. If your only CA replica goes down,
> you cannot issue or renew certificates until it comes back.

> 🔁 **See Module 12** for replica installation and topology management.

[↑ Back to TOC](#table-of-contents)

---

## 5. Directory Information Tree (DIT) Layout

Understanding the LDAP tree helps when troubleshooting or writing custom ACIs.

```mermaid
graph TD
    ROOT["dc=example,dc=com"] --> ACC["cn=accounts"]
    ROOT --> KRB["cn=kerberos"]
    ROOT --> DNS["cn=dns"]
    ROOT --> PKI["cn=pki"]
    ROOT --> SVC["cn=services\n(IPA internal services)"]
    ROOT --> HBAC["cn=hbac"]
    ROOT --> SUDO["cn=sudorules"]
    ROOT --> RBAC["cn=pbac\n(Role-Based Access Control)"]

    ACC --> USERS["cn=users\nuid=jdoe,..."]
    ACC --> GROUPS["cn=groups\ncn=admins,..."]
    ACC --> HOSTS["cn=computers\nfqdn=web01,..."]
    ACC --> SVCPRINC["cn=services\nkrbprincipalname=HTTP/..."]
    ACC --> STAGED["cn=staged users"]
    ACC --> PRESERVED["cn=deleted users"]

    KRB --> REALM["cn=EXAMPLE.COM\nKerberos realm config"]
    PKI --> CATS["cn=cas\nCA objects"]
    PKI --> PROFILES["cn=certificateProfiles"]
```

> 📝 The `cn=accounts` subtree is what SSSD queries. The `cn=kerberos` subtree is
> what the KDC reads. The `cn=dns` subtree is what BIND reads.

[↑ Back to TOC](#table-of-contents)

---

## 6. Replication Overview

FreeIPA uses **multi-master replication** — every replica can accept writes, and
changes propagate to all other replicas automatically.

```mermaid
graph TD
    subgraph "Replication Topology (ring example)"
        A[ipa1\ndc1] <-->|segment| B[ipa2\ndc2]
        B <-->|segment| C[ipa3\ndc3]
        A <-->|segment| C
    end
    subgraph "Data replicated"
        D[389-DS\nall LDAP data]
        E[Dogtag CA\ncert database]
    end
    A --- D
    A --- E
```

Replication has two independent streams:
1. **389-DS replication** — replicates all LDAP data (users, groups, policies, DNS,
   Kerberos principals, etc.)
2. **Dogtag CA replication** — replicates the certificate database between CA replicas
   (uses its own 389-DS instance on port 7389)

Both streams must be healthy. `ipa-healthcheck` monitors both.

> 🔁 **See Module 12** for topology management, adding replicas, and monitoring
> replication status.

[↑ Back to TOC](#table-of-contents)

---

## 7. Lab — Inspect a Running IPA Server

After completing Module 02 (Installation), return here to explore the architecture
hands-on.

```bash
# (server) Show all IPA-managed systemd services and their status
ipactl status

# (server) Show the 389-DS instance name
systemctl list-units 'dirsrv@*'

# (server) Show the Dogtag CA instance
systemctl status pki-tomcatd@pki-tomcat

# (server) Show BIND status
systemctl status named

# (server) List all Kerberos principals in the realm
# (requires valid admin TGT)
kinit admin
ipa service-find --all | grep krbprincipalname

# (server) Explore the LDAP tree root
ldapsearch -Y GSSAPI -b "dc=example,dc=com" -s one "(objectClass=*)" dn

# (server) Show all users container
ldapsearch -Y GSSAPI -b "cn=users,cn=accounts,dc=example,dc=com" \
  -s one "(objectClass=*)" uid

# (server) Show IPA API version and capabilities
ipa ping
ipa env --all | grep -E "(version|domain|realm|host)"

# (server) Show certificate tracking by certmonger
getcert list

# (server) View DNS SRV records for service discovery
dig +short _ldap._tcp.example.com SRV
dig +short _kerberos._tcp.EXAMPLE.COM SRV
dig +short _kerberos._udp.EXAMPLE.COM SRV
```

[↑ Back to TOC](#table-of-contents)
