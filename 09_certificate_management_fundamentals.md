# Module 09 — Certificate Management: Fundamentals
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](./LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![FreeIPA](https://img.shields.io/badge/FreeIPA-v4.12-blue)](https://www.freeipa.org)

> The FreeIPA integrated CA (Dogtag PKI), certificate profiles, requesting and
> managing certificates with `ipa cert-*`, and automating renewal with Certmonger.
>
> 🔁 Certificate topics referenced in other modules:
> - Modules 02, 05: host certs issued during install/enrollment
> - Module 04: PKINIT (cert-based Kerberos auth)
> - Module 10: sub-CAs, ACME, external CA, OCSP, FIPS

## Table of Contents

- [Recommended Background](#recommended-background)
- [Learning Outcomes](#learning-outcomes)
- [1. Dogtag CA Architecture](#1-dogtag-ca-architecture)
  - [1.1 Components](#11-components)
  - [1.2 IPA CA Certificate Chain](#12-ipa-ca-certificate-chain)
- [2. Certificate Profiles](#2-certificate-profiles)
  - [2.1 Built-in Profiles](#21-built-in-profiles)
  - [2.2 Profile Structure](#22-profile-structure)
- [3. Requesting Certificates with ipa cert-request](#3-requesting-certificates-with-ipa-cert-request)
  - [3.1 Generating a CSR](#31-generating-a-csr)
  - [3.2 Requesting a Certificate](#32-requesting-a-certificate)
  - [3.3 Certificate Issuance Decision Path](#33-certificate-issuance-decision-path)
- [4. Managing Certificates](#4-managing-certificates)
  - [4.1 Listing and Viewing Certificates](#41-listing-and-viewing-certificates)
  - [4.2 Revoking Certificates](#42-revoking-certificates)
- [5. Certmonger](#5-certmonger)
  - [5.1 What Certmonger Does](#51-what-certmonger-does)
  - [5.2 Certmonger State Machine](#52-certmonger-state-machine)
  - [5.3 Tracking Certificates](#53-tracking-certificates)
  - [5.4 Renewal Cycle](#54-renewal-cycle)
  - [5.5 Pre-save and Post-save Commands](#55-pre-save-and-post-save-commands)
- [5.6 Renewal Failure Recovery](#56-renewal-failure-recovery)
- [6. IPA Service Certificates](#6-ipa-service-certificates)
  - [6.1 Built-in IPA Certs](#61-built-in-ipa-certs)
  - [6.2 Adding a Certificate to a Custom Service](#62-adding-a-certificate-to-a-custom-service)
- [6.3 Tracking Certificates in NSS Databases](#63-tracking-certificates-in-nss-databases)
- [7. Propagating CA Certificate Changes](#7-propagating-ca-certificate-changes)
- [8. Lab — Certificate Operations](#8-lab--certificate-operations)
- [Key Takeaways](#key-takeaways)


---

## Recommended Background

- Complete Modules 00 through 08.
- Basic familiarity with TLS, CSRs, private keys, and service principals.
- An enrolled host or service available for certificate testing.

## Learning Outcomes

By the end of this module, you should be able to:

- Explain the relationship between IPA, Dogtag CA, and certificate profiles.
- Request, inspect, and revoke certificates with ipa cert-* commands.
- Track certificates with certmonger and understand the renewal lifecycle.
- Operate service certificates safely on both custom services and IPA components.

---

## 1. Dogtag CA Architecture

### 1.1 Components

Dogtag PKI is an enterprise certificate authority embedded in FreeIPA. It runs
as a Tomcat web application and exposes both a REST API and a web admin interface.

```mermaid
graph TD
    subgraph "Dogtag PKI (pki-tomcat)"
        CA["CA Subsystem<br/>:8443/ca/"]
        KRA["KRA Subsystem<br/>Key Recovery Authority<br/>optional"]
    end

    subgraph "IPA Integration"
        API["IPA httpd<br/>:443/ipa/json"] -->|proxy cert requests| CA
        CMON["certmonger<br/>ipa-submit helper"] -->|REST API calls| CA
    end

    subgraph "Storage"
        CA -->|cert database| NSS["("NSS DB<br/>/var/lib/pki/pki-tomcat/alias/")"]
        CA -->|cert/request records| CSDB["("CS 389-DS<br/>localhost:7389")"]
        CSDB <-->|Dogtag replication| REPLICA["("Replica<br/>Dogtag 389-DS")"]
    end

    subgraph "Clients"
        SVC["Service needing cert<br/>httpd, sshd, nginx"] -->|getcert request| CMON
        ADMIN["Admin<br/>ipa cert-request"] -->|IPA API| API
    end
```

### 1.2 IPA CA Certificate Chain

```mermaid
graph TD
    ROOT["IPA CA<br/>(Self-signed root)<br/>Subject: CN=Certificate Authority,O=EXAMPLE.COM<br/>Validity: 20 years<br/>Stored: /etc/ipa/ca.crt"] --> LDAP["LDAP Service Cert<br/>Subject: CN=ipa1.example.com<br/>SAN: ipa1.example.com<br/>Validity: 2 years"]
    ROOT --> HTTP["HTTP Service Cert<br/>Subject: CN=ipa1.example.com<br/>SAN: ipa1.example.com<br/>Validity: 2 years"]
    ROOT --> KDC["KDC Cert<br/>(for PKINIT)<br/>Validity: 2 years"]
    ROOT --> DOGTAG["Dogtag RA Cert<br/>(Registration Authority)<br/>Validity: 2 years"]
    ROOT --> HOST["Host Certs<br/>CN=client01.example.com<br/>Validity: 2 years"]
    ROOT --> SVCERT["Service Certs<br/>CN=HTTP/webapp.example.com<br/>Validity: 2 years"]
    ROOT --> SUBCA["Sub-CAs<br/>(optional)<br/>See Module 10"]
```

[↑ Back to TOC](#table-of-contents)

---

## 2. Certificate Profiles

### 2.1 Built-in Profiles

| Profile ID | Purpose | Key usage |
|------------|---------|-----------|
| `caIPAserviceCert` | Default service/host TLS cert | Digital signature, Key encipherment |
| `IECUserRoles` | User certificates with roles | Digital signature |
| `caServerCert` | Server certificates | Digital signature, Key encipherment |
| `caCACert` | Sub-CA certificates | CA:TRUE, key cert sign |
| `KDCs_PKINIT_Certs` | KDC PKINIT certificate | id-pkinit-KPKdc |
| `acmeIPAServerCert` | ACME-issued server certs | Digital signature, Key encipherment |

```bash
# List all certificate profiles
ipa certprofile-find

# Show a profile's details
ipa certprofile-show caIPAserviceCert

# Show the profile configuration file
ipa certprofile-show caIPAserviceCert --out /tmp/profile.cfg
cat /tmp/profile.cfg
```

### 2.2 Profile Structure

A certificate profile configuration file controls every aspect of the issued cert:

```mermaid
graph TD
    subgraph "Certificate Profile: caIPAserviceCert"
        INP["Input<br/>CMC or PKCS10 CSR"] --> POL
        POL[Policy Components]
        POL --> SAN["SubjectAltName extension<br/>from CSR"]
        POL --> KU["Key Usage:<br/>digitalSignature<br/>keyEncipherment"]
        POL --> EKU["Extended Key Usage:<br/>serverAuth<br/>clientAuth"]
        POL --> VALID["Validity: 731 days<br/>2 years"]
        POL --> SHA[Signature: SHA256withRSA]
        POL --> OUT["Output<br/>PEM certificate"]
    end
```

Key profile configuration parameters:

```ini
# Excerpt from caIPAserviceCert profile
profileId=caIPAserviceCert
desc=Standard profile for IPA service certs

# Validity
policyset.serverCertSet.1.constraint.class_id=noConstraintImpl
policyset.serverCertSet.2.default.class_id=validityDefaultImpl
policyset.serverCertSet.2.default.params.range=731   # days

# Key usage
policyset.serverCertSet.7.default.params.keyUsageCritical=true
policyset.serverCertSet.7.default.params.keyUsageDigitalSignature=true
policyset.serverCertSet.7.default.params.keyUsageKeyEncipherment=true

# Extended key usage
policyset.serverCertSet.8.default.params.exKeyUsageOIDs=1.3.6.1.5.5.7.3.1,1.3.6.1.5.5.7.3.2
# serverAuth, clientAuth
```

[↑ Back to TOC](#table-of-contents)

---

## 3. Requesting Certificates with ipa cert-request

### 3.1 Generating a CSR

Before requesting a certificate from IPA, you need a Certificate Signing Request (CSR):

```bash
# Generate a private key and CSR for a service
# 📝 openssl 3 supports -addext, so a SAN can be added inline without a config file.
#    IPA requires the SAN to match the principal FQDN, so always include it.
openssl req -newkey rsa:4096 -nodes \
  -keyout /etc/pki/tls/private/webapp.key \
  -out /tmp/webapp.csr \
  -subj "/CN=webapp.example.com" \
  -addext "subjectAltName=DNS:webapp.example.com"

# Or use a config file to include SANs
cat > /tmp/webapp-openssl.cnf << 'EOF'
[req]
default_bits = 4096
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[dn]
CN = webapp.example.com

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = webapp.example.com
DNS.2 = webapp
IP.1 = 192.168.1.25
EOF

openssl req -newkey rsa:4096 -nodes \
  -keyout /etc/pki/tls/private/webapp.key \
  -out /tmp/webapp.csr \
  -config /tmp/webapp-openssl.cnf

# Verify the CSR
openssl req -verify -in /tmp/webapp.csr -noout -text | grep -A5 "Subject Alternative"
```

### 3.2 Requesting a Certificate

```bash
# Request a certificate for a service principal
kinit admin
ipa cert-request /tmp/webapp.csr \
  --principal=HTTP/webapp.example.com \
  --profile-id=caIPAserviceCert \
  --certificate-out=/etc/pki/tls/certs/webapp.crt

# Verify the issued certificate
openssl x509 -in /etc/pki/tls/certs/webapp.crt -text -noout \
  | grep -E "(Subject:|Issuer:|Not Before|Not After|DNS:)"

# Request a user certificate
ipa cert-request /tmp/jdoe.csr \
  --principal=jdoe \
  --profile-id=IECUserRoles \
  --certificate-out=/home/jdoe/jdoe.crt
```

### 3.3 Certificate Issuance Decision Path

```mermaid
flowchart TD
    A["ipa cert-request CSR<br/>--principal=..."] --> B["IPA API validates:<br/>Does principal exist?"]
    B -->|No| FAIL1[Error: principal not found]
    B -->|Yes| C["Does profile exist<br/>and is enabled?"]
    C -->|No| FAIL2[Error: profile not found]
    C -->|Yes| D["Does caller have permission?<br/>System: Request Certificate"]
    D -->|No| FAIL3[Error: permission denied]
    D -->|Yes| E["Validate CSR:<br/>SAN matches principal FQDN?"]
    E -->|No - SAN mismatch| FAIL4["Error: Subject alt name<br/>does not match principal"]
    E -->|Yes| F[Forward CSR to Dogtag CA]
    F --> G["Dogtag applies profile policies:<br/>Validity, Key usage, SANs, Signature alg"]
    G --> H{"Profile constraints<br/>satisfied?"}
    H -->|No| FAIL5[Error: profile constraint violation]
    H -->|Yes| I[Sign certificate with IPA CA key]
    I --> J["Store cert serial in LDAP<br/>+ Dogtag internal DB"]
    J --> K[Return PEM certificate]
```

[↑ Back to TOC](#table-of-contents)

---

## 4. Managing Certificates

### 4.1 Listing and Viewing Certificates

```bash
# List all certificates issued by IPA CA
ipa cert-find

# Filter by principal
ipa cert-find --subject=webapp.example.com

# Filter by validity
ipa cert-find --validnotafter-from=2026-01-01

# Show a specific certificate (by serial number)
ipa cert-show 42

# Show certificate in PEM format
ipa cert-show 42 --certificate-out=/tmp/cert42.pem
openssl x509 -in /tmp/cert42.pem -text -noout

# Find certificates expiring within 30 days
ipa cert-find --validnotafter-to=$(date -d "+30 days" +%Y-%m-%d)

# Show all certs for a host
ipa host-show web01.example.com --all | grep -i "certificate:"
```

### 4.2 Revoking Certificates

```mermaid
sequenceDiagram
    participant ADM as Admin
    participant API as IPA API
    participant CA as Dogtag CA
    participant CRL as CRL/OCSP

    ADM->>API: ipa cert-revoke <serial><br/>--revocation-reason=4
    API->>CA: REST: revoke certificate serial N<br/>reason: Superseded (4)
    CA->>CA: Mark cert as revoked<br/>in Dogtag DB
    CA->>CRL: Update CRL (Certificate Revocation List)<br/>(regenerated periodically)
    CA->>CRL: OCSP responder reflects revocation immediately
    CA-->>API: Revocation confirmed
    API-->>ADM: Certificate revoked
```

**Revocation reason codes:**

| Code | Reason | Use case |
|------|--------|---------|
| 0 | Unspecified | General revocation |
| 1 | Key Compromise | Private key exposed |
| 2 | CA Compromise | CA key exposed |
| 3 | Affiliation Changed | User moved to different org |
| 4 | Superseded | Cert replaced by new one |
| 5 | Cessation of Operation | Service decommissioned |
| 6 | Certificate Hold | Temporary (can be restored) |
| 8 | Remove from CRL | Remove a held cert from the CRL (delta-CRL only) |
| 9 | Privilege Withdrawn | Subject no longer authorised for the cert's purpose |
| 10 | AA Compromise | Attribute authority key exposed |

> 📝 Per RFC 5280 the value `7` is unused. Reason `8` (removeFromCRL) only
> applies to a certificate previously placed on hold (code `6`).

```bash
# Revoke a certificate by serial number
ipa cert-revoke 42 --revocation-reason=4

# Find and revoke all certs for a decommissioned service
ipa cert-find --subject=oldservice.example.com \
  | grep "Serial number:" \
  | awk '{print $3}' \
  | while read serial; do
      ipa cert-revoke "$serial" --revocation-reason=5
    done

# Check revocation status with OCSP
# 📝 -noverify skips verification of the OCSP responder's signing cert, which
#    keeps the check consistent with the other OCSP examples in this course.
openssl ocsp \
  -issuer /etc/ipa/ca.crt \
  -cert /etc/pki/tls/certs/webapp.crt \
  -url http://ipa1.example.com/ca/ocsp \
  -noverify
```

[↑ Back to TOC](#table-of-contents)

---

## 5. Certmonger

### 5.1 What Certmonger Does

Certmonger is a daemon that **requests, stores, tracks, and auto-renews** certificates.
It runs on both the IPA server (managing IPA's own service certs) and on all
enrolled clients (managing host and service certs).

```mermaid
graph TD
    CMON[certmonger daemon] --> TRACK["("Tracking entries<br/>/var/lib/certmonger/requests/")"]
    CMON -->|submit helper: ipa-submit| API[IPA API → Dogtag CA]
    CMON -->|writes certs| FS["Certificate files<br/>/etc/pki/tls/certs/<br/>/etc/ipa/<br/>NSS databases"]
    CMON -->|pre-save command| PRE["Pre-save hook<br/>e.g., stop httpd"]
    CMON -->|post-save command| POST["Post-save hook<br/>e.g., start httpd, reload nginx"]
    TIMER["systemd timer<br/>certmonger-renewal.timer"] -->|triggers renewal check| CMON
```

### 5.2 Certmonger State Machine

Each tracked certificate moves through a state machine:

```mermaid
stateDiagram-v2
    [*] --> MONITORING: getcert request<br/>cert issued
    MONITORING --> NEED_TO_ENROLL: new request<br/>no cert yet
    NEED_TO_ENROLL --> SUBMITTING: submit to CA
    SUBMITTING --> NEED_CA_NAME: CA unknown
    SUBMITTING --> CA_UNREACHABLE: network error
    SUBMITTING --> CA_REJECTED: CA denied request
    SUBMITTING --> NEED_CSR_REJECTED: CSR problem
    SUBMITTING --> POST_SAVED_CERT: cert issued
    POST_SAVED_CERT --> MONITORING: post-save cmd ran OK
    MONITORING --> NEED_TO_SUBMIT: cert near expiry<br/>(renewal window reached)
    NEED_TO_SUBMIT --> SUBMITTING: renewal submit
    CA_UNREACHABLE --> SUBMITTING: retry after backoff
    CA_REJECTED --> [*]: manual intervention needed
```

| State | Meaning |
|-------|---------|
| `MONITORING` | Cert is valid, certmonger is watching expiry |
| `NEED_TO_SUBMIT` | Cert is within renewal window (default: 28 days before expiry) |
| `SUBMITTING` | Request sent to CA, waiting for response |
| `CA_UNREACHABLE` | Cannot reach CA, will retry |
| `CA_REJECTED` | CA refused the request (check logs) |
| `POST_SAVED_CERT` | Cert saved, running post-save command |

```bash
# View all tracked certificates and their states
getcert list

# Compact view showing only state and subject
getcert list | grep -E "(Request ID|status|subject)"

# View a specific tracking entry
getcert list -i 20240101120000   # by request ID

# Check for any non-MONITORING states (potential problems)
getcert list | grep -v "status: MONITORING"
```

### 5.3 Tracking Certificates

```bash
# Track an existing certificate file (already have key + cert)
getcert start-tracking \
  -f /etc/pki/tls/certs/webapp.crt \
  -k /etc/pki/tls/private/webapp.key \
  -p HTTP/webapp.example.com \
  -c IPA

# Request a NEW certificate and start tracking it
getcert request \
  -f /etc/pki/tls/certs/myservice.crt \
  -k /etc/pki/tls/private/myservice.key \
  -N CN=myservice.example.com \
  -D myservice.example.com \
  -K host/myservice.example.com \
  -T caIPAserviceCert \
  -C "systemctl reload nginx"    # post-save command

# Track a certificate in an NSS database
getcert request \
  -d /etc/dirsrv/slapd-EXAMPLE-COM \
  -n "Server-Cert" \
  -K ldap/ipa1.example.com \
  -T caIPAserviceCert

# Stop tracking a certificate (does NOT revoke it)
getcert stop-tracking -f /etc/pki/tls/certs/webapp.crt
```

### 5.4 Renewal Cycle

```mermaid
sequenceDiagram
    participant TIMER as systemd timer<br/>(certmonger-renewal)
    participant CMON as certmonger daemon
    participant IPA as IPA API
    participant CA as Dogtag CA
    participant SVC as Service (nginx)

    TIMER->>CMON: periodic renewal check

    CMON->>CMON: Check each tracked cert:<br/>cert expires in < 28 days?

    Note over CMON: webapp.crt expires in 15 days → renew

    CMON->>CMON: Run pre-save command<br/>(if configured)

    CMON->>IPA: Submit renewal CSR<br/>(ipa-submit helper)
    IPA->>CA: Forward CSR
    CA-->>IPA: New certificate
    IPA-->>CMON: New certificate PEM

    CMON->>CMON: Write new cert to<br/>/etc/pki/tls/certs/webapp.crt

    CMON->>SVC: Run post-save command:<br/>systemctl reload nginx

    SVC-->>CMON: Exit 0

    CMON->>CMON: Update state → MONITORING<br/>Update expiry date in tracking entry
```

### 5.5 Pre-save and Post-save Commands

```bash
# Set a post-save command on an existing tracking entry
getcert start-tracking \
  -f /etc/pki/tls/certs/webapp.crt \
  -k /etc/pki/tls/private/webapp.key \
  -c IPA \
  -C "systemctl reload nginx"    # runs after cert is saved

# Example post-save commands for common services:
# Apache: systemctl reload httpd
# nginx: systemctl reload nginx
# HAProxy: systemctl reload haproxy
# 389-DS: restart-dirsrv EXAMPLE-COM (done by IPA automatically)
# Dogtag: pki-server subsystem-enable ca (done by IPA automatically)

# Force immediate renewal (don't wait for expiry window)
getcert resubmit -f /etc/pki/tls/certs/webapp.crt

# Resubmit by request ID
getcert resubmit -i 20240101120000
```

### 5.6 Renewal Failure Recovery

If a tracked certificate enters the renewal window but does not return to `MONITORING`, treat it like an incident before the certificate actually expires.

```bash
# 1. Inspect the failing request in detail
getcert list -i 20240101120000

# 2. Retry submission and restart certmonger if the helper is stuck
getcert resubmit -i 20240101120000
systemctl restart certmonger

# 3. Review helper and CA errors
journalctl -u certmonger --since "1 hour ago"
journalctl -u pki-tomcatd@pki-tomcat --since "1 hour ago"

# 4. If the certificate is already expired, request a replacement immediately
getcert request -f /etc/pki/tls/certs/webapp.crt \
  -k /etc/pki/tls/private/webapp.key \
  -N CN=webapp.example.com \
  -D webapp.example.com \
  -K HTTP/webapp.example.com \
  -T caIPAserviceCert \
  -C "systemctl reload httpd"
```

> Do not assume service deletion revokes certificates automatically. Revoke compromised or abandoned certificates explicitly with `ipa cert-revoke` and then reload the affected service.

[↑ Back to TOC](#table-of-contents)

---

## 6. IPA Service Certificates

### 6.1 Built-in IPA Certs

The IPA server itself has several certificates managed by certmonger:

```bash
# View all certs tracked by certmonger on the IPA server
getcert list

# Typical tracked certs on an IPA server:
# /var/lib/ipa/certs/httpd.crt   - HTTPD (Web UI + API) cert  (mod_ssl PEM)
# /var/lib/ipa/private/httpd.key - HTTPD private key          (mod_ssl PEM)
# /etc/dirsrv/slapd-EXAMPLE-COM  - LDAP service cert (NSS DB)
# /var/lib/ipa/ra-agent.pem      - Dogtag RA agent cert
# /var/kerberos/krb5kdc/kdc.crt  - KDC PKINIT cert
```

> 📝 FreeIPA 4.12 on RHEL 10 serves the Web UI/API through `mod_ssl` using PEM
> files (`/var/lib/ipa/certs/httpd.crt` + `/var/lib/ipa/private/httpd.key`).
> The legacy `/etc/httpd/alias` NSS database (mod_nss) is no longer used.

```mermaid
graph TD
    subgraph "IPA Server Certmonger-tracked Certs"
        H["httpd<br/>HTTPS UI+API<br/>HTTP/ipa1.example.com"] -->|cert| HC["var/lib/ipa/certs/httpd.crt<br/>mod_ssl PEM"]
        L["389-DS<br/>LDAPS<br/>ldap/ipa1.example.com"] -->|cert| LC["etc/dirsrv/slapd-EXAMPLE-COM<br/>NSS DB"]
        K["krb5kdc<br/>PKINIT<br/>krbprinc/ipa1.example.com"] -->|cert| KC["var/kerberos/krb5kdc<br/>kdc.crt"]
        R["Dogtag RA Agent<br/>registration authority"] -->|cert| RC["var/lib/ipa<br/>ra-agent.pem"]
    end
```

### 6.2 Adding a Certificate to a Custom Service

```bash
# (server) Create the service principal first
ipa service-add HTTP/webapp.example.com

# (client) Request and track the certificate using certmonger
getcert request \
  -f /etc/pki/tls/certs/webapp.crt \
  -k /etc/pki/tls/private/webapp.key \
  -N CN=webapp.example.com \
  -D webapp.example.com \
  -K HTTP/webapp.example.com \
  -T caIPAserviceCert \
  -C "systemctl reload httpd"

# Wait for issuance (or force)
getcert list -f /etc/pki/tls/certs/webapp.crt
# Status should change: SUBMITTING → POST_SAVED_CERT → MONITORING

# Verify the certificate
openssl x509 -in /etc/pki/tls/certs/webapp.crt -text -noout \
  | grep -E "(Subject:|Issuer:|Not Before|Not After)"
```

### 6.3 Tracking Certificates in NSS Databases

Many built-in IPA services store certificates in NSS databases instead of PEM files. Track those entries directly so renewal automation matches the actual service storage format.

> 📝 IPA already tracks its own system certificates (the 389-DS `Server-Cert`,
> the httpd cert, the RA agent, the KDC PKINIT cert, etc.) via certmonger. Do
> NOT add duplicate tracking for IPA-managed certs — the example below applies
> to YOUR OWN service that happens to use an NSS database. The `slapd-EXAMPLE-COM`
> path is shown only to illustrate the NSS-tracking syntax.

> ⚠️ Restarting `dirsrv@EXAMPLE-COM.service` restarts the IPA Directory Server,
> which interrupts the entire IPA stack (LDAP, Kerberos KDC backend, the Web UI,
> and any service depending on 389-DS). For IPA's own `Server-Cert`, IPA handles
> renewal and reload automatically — never wire a manual restart into it.

```bash
# Track the 389-DS Server-Cert entry in the directory server NSS database
getcert start-tracking \
  -d /etc/dirsrv/slapd-EXAMPLE-COM \
  -n Server-Cert \
  -C "systemctl restart dirsrv@EXAMPLE-COM.service"

# Inspect the tracking entry
getcert list -d /etc/dirsrv/slapd-EXAMPLE-COM -n Server-Cert

# Force a renewal test when you need to validate the workflow
getcert resubmit -d /etc/dirsrv/slapd-EXAMPLE-COM -n Server-Cert
```

> NSS-backed services usually need a restart or service-specific reload after renewal because the certificate nickname stays the same while the underlying object changes.

[↑ Back to TOC](#table-of-contents)

---

## 7. Propagating CA Certificate Changes

When the IPA CA certificate is renewed or replaced, all enrolled clients must
receive the updated CA cert chain.

```bash
# (server) Renew/update the CA cert chain
ipa-cacert-manage renew

# (server) After renewal, update all server components
ipa-certupdate

# (client) Update a client's CA cert trust store
ipa-certupdate   # run on each client

# (server) Check CA cert expiry
ipa-cacert-manage list

# (server) Check all tracked certs + CA expiry via healthcheck
ipa-healthcheck --source ipahealthcheck.ipa.certs --output-type human
```

[↑ Back to TOC](#table-of-contents)

---

## 8. Lab — Certificate Operations

```bash
# ── SETUP ────────────────────────────────────────────────────────────────────

kinit admin

# Create a service principal for our test web app
ipa service-add HTTP/webapp.example.com

# ── STEP 1: Manual certificate request ──────────────────────────────────────

# Generate private key + CSR
mkdir -p /etc/pki/tls/{certs,private}
openssl req -newkey rsa:4096 -nodes \
  -keyout /etc/pki/tls/private/webapp.key \
  -out /tmp/webapp.csr \
  -subj "/CN=webapp.example.com"

chmod 600 /etc/pki/tls/private/webapp.key

# Request the certificate from IPA
ipa cert-request /tmp/webapp.csr \
  --principal=HTTP/webapp.example.com \
  --profile-id=caIPAserviceCert \
  --certificate-out=/etc/pki/tls/certs/webapp.crt

# Verify it
openssl x509 -in /etc/pki/tls/certs/webapp.crt -text -noout \
  | grep -E "(Subject:|Issuer:|Not Before|Not After|DNS:)"

# ── STEP 2: Certmonger tracking ──────────────────────────────────────────────

# Track the certificate we just issued
getcert start-tracking \
  -f /etc/pki/tls/certs/webapp.crt \
  -k /etc/pki/tls/private/webapp.key \
  -K HTTP/webapp.example.com \
  -c IPA \
  -C "echo 'Cert renewed at $(date)' >> /tmp/cert-renewal.log"

# View tracking status
getcert list -f /etc/pki/tls/certs/webapp.crt

# ── STEP 3: Force renewal ────────────────────────────────────────────────────

# Force an immediate renewal
getcert resubmit -f /etc/pki/tls/certs/webapp.crt
sleep 5
getcert list -f /etc/pki/tls/certs/webapp.crt
# Status should cycle through SUBMITTING → POST_SAVED_CERT → MONITORING

# Check the post-save command ran
cat /tmp/cert-renewal.log

# ── STEP 4: View and revoke ───────────────────────────────────────────────────

# Get the certificate serial number
SERIAL=$(openssl x509 -in /etc/pki/tls/certs/webapp.crt -noout -serial \
  | awk -F= '{print $2}' \
  | xargs printf "%d\n")
echo "Serial (decimal): $SERIAL"

# View the cert in IPA
ipa cert-show $SERIAL

# Revoke it (superseded)
ipa cert-revoke $SERIAL --revocation-reason=4

# Check OCSP
openssl ocsp \
  -issuer /etc/ipa/ca.crt \
  -cert /etc/pki/tls/certs/webapp.crt \
  -url http://ipa1.example.com/ca/ocsp \
  -noverify

# ── STEP 5: Health check ──────────────────────────────────────────────────────

ipa-healthcheck \
  --source ipahealthcheck.ipa.certs \
  --output-type human \
  2>&1 | grep -v SUCCESS

# Stop tracking
getcert stop-tracking -f /etc/pki/tls/certs/webapp.crt

# Clean up
ipa service-del HTTP/webapp.example.com
```

> 🔁 **Next:** Continue with [Module 10 — Certificate Management: Advanced](10_certificate_management_advanced.md)
> to explore sub-CAs, custom profiles, OCSP, CRL management, external CA chaining,
> certmonger hooks, and FIPS-mode constraints.


---

## Key Takeaways

- Certificate issuance is policy-driven, not just a signing operation.
- Certmonger is the operational glue that turns certificate issuance into renewal automation.
- Post-save hooks matter because many services must reload to use new certs.
- Recovery guidance is essential when automated renewal does not return to MONITORING.

[↑ Back to TOC](#table-of-contents)

---

*Licensed under [CC BY-NC-SA 4.0](LICENSE.md) · © 2026 UncleJS*