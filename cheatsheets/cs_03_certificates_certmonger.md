# CS-03 — Certificates and Certmonger Cheatsheet
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](../LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![FreeIPA](https://img.shields.io/badge/FreeIPA-v4.12-blue)](https://www.freeipa.org)

> Quick-reference for certificate operations, certmonger management, Dogtag CA, and CSR workflows in FreeIPA on RHEL 10.

---

> 🔁 **See also:** [Module 09 — Certificate Management: Fundamentals](../09_certificate_management_fundamentals.md) · [Module 10 — Certificate Management: Advanced](../10_certificate_management_advanced.md)

---

## Table of Contents

- [Certmonger Quick Reference](#certmonger-quick-reference)
- [Request and Renew Certificates](#request-and-renew-certificates)
- [IPA cert-* Commands](#ipa-cert--commands)
- [Certificate Profiles](#certificate-profiles)
- [CSR Operations](#csr-operations)
- [CA Certificate and Chain](#ca-certificate-and-chain)
- [OCSP and CRL](#ocsp-and-crl)
- [Dogtag CA Operations](#dogtag-ca-operations)
- [Sub-CAs](#sub-cas)
- [Troubleshooting](#troubleshooting)

---

## Certmonger Quick Reference

```bash
# List all tracked certificates
sudo getcert list

# Compact output — status + expiry only
sudo getcert list | grep -E "Request ID|status|expires"

# Status meanings:
# MONITORING          = cert valid, being watched for renewal
# NEED_CSR_GEN        = about to generate a CSR
# CA_WORKING          = CSR submitted, waiting for CA
# CA_UNREACHABLE      = can't reach CA
# CA_REJECTED         = CA refused the request
# NEED_CA             = no CA configured for this request
# POST_SAVED_CERT     = cert saved, running post-save command

# Force immediate renewal attempt
sudo getcert resubmit -i <request-id>
sudo getcert resubmit -f /path/to/cert.crt

# Stop tracking a certificate (won't delete files)
sudo getcert stop-tracking -i <request-id>
sudo getcert stop-tracking -f /path/to/cert.crt

# Start tracking an existing cert
sudo getcert start-tracking \
    -f /etc/httpd/conf/server.crt \
    -k /etc/httpd/conf/server.key \
    -c IPA

# List CAs configured in certmonger
sudo getcert list-cas
```

[↑ Back to TOC](#table-of-contents)

---

## Request and Renew Certificates

```bash
# Request cert for an IPA service principal (certmonger)
sudo getcert request \
    -K HTTP/webapp.example.com \
    -f /etc/httpd/conf/server.crt \
    -k /etc/httpd/conf/server.key \
    -w                           # wait for cert

# With SANs
sudo getcert request \
    -K HTTP/webapp.example.com \
    -f /etc/httpd/conf/server.crt \
    -k /etc/httpd/conf/server.key \
    -D webapp.example.com \
    -D webapp-alt.example.com \
    -w

# Request with specific profile
sudo getcert request \
    -K host/server1.example.com \
    -T caIPAserviceCert \
    -f /etc/ssl/server.crt \
    -k /etc/ssl/server.key \
    -w

# Request with post-save command (e.g., restart httpd)
sudo getcert request \
    -K HTTP/webapp.example.com \
    -f /etc/httpd/conf/server.crt \
    -k /etc/httpd/conf/server.key \
    -C "systemctl reload httpd" \
    -w

# Request cert for a user principal
sudo getcert request \
    -K jsmith@EXAMPLE.COM \
    -T caIPAuserCert \
    -f /tmp/jsmith.crt \
    -k /tmp/jsmith.key \
    -w

# Manual CSR-based request (when not using certmonger)
# Note: RSA-2048 is the minimum; use RSA-3072+ for new keys (required in FIPS mode)
openssl genrsa -out server.key 3072
openssl req -new -key server.key \
    -subj "/CN=server1.example.com" \
    -out server.csr

ipa cert-request server.csr \
    --principal=host/server1.example.com \
    --profile-id=caIPAserviceCert
```

[↑ Back to TOC](#table-of-contents)

---

## IPA cert-* Commands

```bash
# Request certificate using IPA CLI
ipa cert-request \
    --principal=HTTP/webapp.example.com \
    --profile-id=caIPAserviceCert \
    server.csr

# Show certificate by serial number
ipa cert-show 15
ipa cert-show 15 --out=/tmp/cert-15.pem

# Find certificates
ipa cert-find
ipa cert-find --subject=webapp.example.com
ipa cert-find --issuer="CN=Certificate Authority"
ipa cert-find --revocation-reason=0     # all valid (not revoked)
ipa cert-find \
    --validnotafter-from=20260101000000Z \
    --validnotafter-to=20261231235959Z

# Revoke a certificate
ipa cert-revoke 15 --revocation-reason=1   # 0=unspecified, 1=key compromise

# Revocation reasons:
# 0 = Unspecified
# 1 = Key Compromise
# 2 = CA Compromise
# 3 = Affiliation Changed
# 4 = Superseded
# 5 = Cessation of Operation
# 6 = Certificate Hold (temporary)
# 8 = Remove from Hold (unrevoke reason=6)

# Unrevoke a certificate (reason 6 only)
ipa cert-remove-hold 15

# Remove certificate from IPA tracking (does not revoke)
ipa cert-del <serial>
```

[↑ Back to TOC](#table-of-contents)

---

## Certificate Profiles

```bash
# List available profiles
ipa certprofile-find

# Built-in profiles:
# caIPAserviceCert     — Service certificates (httpd, etc.)
# caIPAuserCert        — User certificates
# KDCs_PKINIT_Certs    — KDC PKINIT
# acmeIPAServerCert    — ACME-issued certs

# Show profile configuration
ipa certprofile-show caIPAserviceCert
ipa certprofile-show caIPAserviceCert --out=/tmp/service-profile.cfg

# Import a custom profile
ipa certprofile-import MyCustomProfile \
    --file=/tmp/my-profile.cfg \
    --desc="Custom server cert profile" \
    --store=true

# Modify profile (download, edit, re-import)
ipa certprofile-show caIPAserviceCert --out=/tmp/svc.cfg
# Edit /tmp/svc.cfg (e.g., change validity period)
ipa certprofile-mod caIPAserviceCert --file=/tmp/svc.cfg

# Delete custom profile (cannot delete built-ins)
ipa certprofile-del MyCustomProfile
```

[↑ Back to TOC](#table-of-contents)

---

## CSR Operations

```bash
# Generate RSA key + CSR (use 3072+ for new keys; 2048 is the absolute minimum)
openssl genrsa -out server.key 3072
openssl req -new -key server.key \
    -subj "/CN=server1.example.com/O=EXAMPLE.COM" \
    -out server.csr

# Generate with SANs (via openssl config)
openssl req -new -key server.key \
    -subj "/CN=server1.example.com" \
    -reqexts SAN \
    -config <(cat /etc/pki/tls/openssl.cnf; printf "[SAN]\nsubjectAltName=DNS:server1.example.com,DNS:server1") \
    -out server.csr

# Generate ECDSA key + CSR (preferred for new deployments)
openssl ecparam -name prime256v1 -genkey -noout -out ec.key
openssl req -new -key ec.key \
    -subj "/CN=server1.example.com" \
    -out ec.csr

# Inspect CSR
openssl req -in server.csr -noout -text
openssl req -in server.csr -noout -subject -verify

# Verify cert matches key
openssl x509 -in server.crt -noout -modulus | md5sum
openssl rsa  -in server.key -noout -modulus | md5sum
# Both md5sums must match

# Inspect issued certificate
openssl x509 -in server.crt -noout -text
openssl x509 -in server.crt -noout -subject -issuer -dates -fingerprint -sha256
```

[↑ Back to TOC](#table-of-contents)

---

## CA Certificate and Chain

```bash
# View IPA CA certificate
openssl x509 -in /etc/ipa/ca.crt -noout -text
openssl x509 -in /etc/ipa/ca.crt -noout -subject -issuer -dates

# Get CA cert fingerprint (for verification)
openssl x509 -in /etc/ipa/ca.crt -noout -fingerprint -sha256

# Add IPA CA to system trust store
sudo cp /etc/ipa/ca.crt \
    /etc/pki/ca-trust/source/anchors/ipa-ca.crt
sudo update-ca-trust extract

# Verify certificate against IPA CA
openssl verify -CAfile /etc/ipa/ca.crt server.crt

# Get full chain
ipa ca-show ipa 2>/dev/null || \
    openssl s_client -connect ipa1.example.com:443 </dev/null 2>/dev/null | \
    openssl x509 -noout -issuer

# Download CA certificate from IPA
curl -sk https://ipa1.example.com/ipa/config/ca.crt \
    -o /tmp/ipa-ca.crt
openssl x509 -in /tmp/ipa-ca.crt -noout -subject -dates
```

[↑ Back to TOC](#table-of-contents)

---

## OCSP and CRL

```bash
# Check certificate via OCSP
# Get OCSP URI from cert
OCSP_URI=$(openssl x509 -in server.crt -noout -ocsp_uri)
echo "OCSP: $OCSP_URI"

# Query OCSP
openssl ocsp \
    -issuer /etc/ipa/ca.crt \
    -cert server.crt \
    -url "$OCSP_URI" \
    -resp_text | grep -E "Response Status|Cert Status"

# Download and inspect CRL
curl -s http://ipa1.example.com/ipa/crl/MasterCRL.bin \
    -o /tmp/ipa.crl
openssl crl -inform DER -in /tmp/ipa.crl \
    -noout -lastupdate -nextupdate -issuer

# Check if a cert is in the CRL
openssl crl -inform DER -in /tmp/ipa.crl \
    -text | grep -A3 "Serial Number: $(openssl x509 -in server.crt -noout -serial | cut -d= -f2)"

# CRL master info
ipa config-show | grep "CA renewal master"
sudo grep "crl.MasterCRL.enable" \
    /etc/pki/pki-tomcat/ca/CS.cfg
```

[↑ Back to TOC](#table-of-contents)

---

## Dogtag CA Operations

```bash
# Check Dogtag CA status
sudo systemctl status pki-tomcatd@pki-tomcat.service

# Dogtag log files
sudo ls -lh /var/log/pki/pki-tomcat/ca/
sudo tail -30 /var/log/pki/pki-tomcat/ca/system
sudo tail -30 /var/log/pki/pki-tomcat/ca/transactions

# Dogtag CA URL
# Agent UI: https://ipa1.example.com:8443/ca/agent/ca/
# End-entity UI: https://ipa1.example.com:8443/ca/ee/ca/

# Check CA is responding
curl -sk https://ipa1.example.com/ca/ee/ca/ | head -5
curl -sk https://ipa1.example.com:8443/ca/admin/ca/getStatus

# Restart Dogtag
sudo systemctl restart pki-tomcatd@pki-tomcat.service

# Dogtag config directory
ls /etc/pki/pki-tomcat/ca/

# Key CS.cfg settings
sudo grep -E "^ca\.(crl|signing|subsystem)" \
    /etc/pki/pki-tomcat/ca/CS.cfg | head -20
```

[↑ Back to TOC](#table-of-contents)

---

## Sub-CAs

```bash
# List sub-CAs
ipa ca-find

# Create a sub-CA
ipa ca-add DevCA \
    --subject="CN=DevCA,O=EXAMPLE.COM" \
    --desc="Sub-CA for development certificates"

# Show sub-CA
ipa ca-show DevCA

# Show sub-CA certificate
ipa ca-show DevCA --certificate-out=/tmp/devca.pem
openssl x509 -in /tmp/devca.pem -noout -subject -dates

# Create profile scoped to a sub-CA
ipa certprofile-add DevServerCert \
    --file=/tmp/dev-profile.cfg \
    --desc="Dev server cert via DevCA" \
    --store=true

# Request certificate from a specific sub-CA
ipa cert-request server.csr \
    --principal=HTTP/devapp.example.com \
    --profile-id=DevServerCert \
    --ca=DevCA

# Disable a sub-CA
ipa ca-disable DevCA

# Delete a sub-CA (revokes its certificate)
ipa ca-del DevCA
```

[↑ Back to TOC](#table-of-contents)

---

## Troubleshooting

```bash
# Cert not renewing — check certmonger request status
sudo getcert list -v | grep -A10 "Request ID:"

# Force resubmit
sudo getcert resubmit -i <id>

# CA unreachable
sudo systemctl status pki-tomcatd@pki-tomcat.service
curl -sk https://ipa1.example.com/ca/ee/ca/ | head -3

# CA rejected — check profile and principal
ipa certprofile-show caIPAserviceCert | head -20
ipa service-show HTTP/webapp.example.com

# Certificate verify fails (TLS)
openssl verify -CAfile /etc/ipa/ca.crt server.crt
# If "unable to get local issuer certificate" → add CA to trust store

# Cert expired and services won't start
sudo getcert list | grep "status:"
sudo getcert resubmit -f /var/lib/ipa/certs/httpd.crt
sudo systemctl status certmonger
sudo journalctl -u certmonger --since "1 hour ago"

# View certmonger debug log
sudo journalctl -u certmonger --since "today" | grep -iE "error|fail|warn"

# Check IPA CA healthcheck
sudo ipa-healthcheck --source ipahealthcheck.ipa.certs
```

[↑ Back to TOC](#table-of-contents)

---

*Licensed under [CC BY-NC-SA 4.0](../LICENSE.md) · © 2026 UncleJS*