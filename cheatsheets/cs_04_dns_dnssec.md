# CS-04 — DNS and DNSSEC Cheatsheet
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](../LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![FreeIPA](https://img.shields.io/badge/FreeIPA-v4.12-blue)](https://www.freeipa.org)

> Quick-reference for DNS zone management, record operations, DNSSEC, and forwarders in FreeIPA on RHEL 10.

---

> 🔁 **See also:** [Module 06 — DNS & DNSSEC](../06_dns_dnssec.md)

---

## Table of Contents

- [Zone Management](#zone-management)
- [Record Operations](#record-operations)
- [Reverse Zones](#reverse-zones)
- [DNS Forwarders](#dns-forwarders)
- [DNSSEC Operations](#dnssec-operations)
- [SRV Records](#srv-records)
- [DNS Diagnostics](#dns-diagnostics)
- [named Service](#named-service)
- [Troubleshooting](#troubleshooting)

---

## Zone Management

```bash
# List all DNS zones
ipa dnszone-find

# Show zone details
ipa dnszone-show example.com

# Add a DNS zone
ipa dnszone-add newzone.example.com \
    --name-server=ipa1.example.com. \
    --admin-email=dnsadmin@example.com

# Add zone with dynamic update settings
ipa dnszone-add newzone.example.com \
    --name-server=ipa1.example.com. \
    --dynamic-update=true \
    --allow-sync-ptr=true

# Modify zone settings
ipa dnszone-mod example.com \
    --ttl=86400 \
    --default-ttl=3600 \
    --refresh=3600 \
    --retry=900 \
    --expire=604800 \
    --minimum=86400

# Disable a zone (stops serving, keeps data)
ipa dnszone-disable example.com

# Enable a zone
ipa dnszone-enable example.com

# Delete a zone (and all its records)
ipa dnszone-del oldzone.example.com
```

[↑ Back to TOC](#table-of-contents)

---

## Record Operations

```bash
# Add A record
ipa dnsrecord-add example.com server2 \
    --a-rec=192.168.1.21

# Add A record with TTL
ipa dnsrecord-add example.com server2 \
    --a-rec=192.168.1.21 \
    --a-ttl=300

# Add AAAA record (IPv6)
ipa dnsrecord-add example.com server2 \
    --aaaa-rec=2001:db8::21

# Add CNAME
ipa dnsrecord-add example.com www \
    --cname-rec=webserver.example.com.

# Add MX record
ipa dnsrecord-add example.com @ \
    --mx-rec="10 mail.example.com."

# Add TXT record (e.g., SPF)
ipa dnsrecord-add example.com @ \
    --txt-rec="v=spf1 ip4:192.168.1.0/24 ~all"

# Add SRV record
ipa dnsrecord-add example.com _custom._tcp \
    --srv-rec="0 100 8080 appserver.example.com."

# Add NS record
ipa dnsrecord-add example.com subdomain \
    --ns-rec=ns1.example.com.

# Show records for a name
ipa dnsrecord-show example.com server2

# Find all records in a zone
ipa dnsrecord-find example.com
ipa dnsrecord-find example.com --name=server2
# Note: dnsrecord-find has no --type filter; use typed option to narrow results:
ipa dnsrecord-find example.com --a-rec=""

# Modify a record
ipa dnsrecord-mod example.com server2 \
    --a-rec=192.168.1.25

# Delete a specific record
ipa dnsrecord-del example.com server2 \
    --a-rec=192.168.1.21

# Delete all records for a name
ipa dnsrecord-del example.com oldserver \
    --del-all
```

[↑ Back to TOC](#table-of-contents)

---

## Reverse Zones

```bash
# Add reverse zone (for 192.168.1.x)
ipa dnszone-add 1.168.192.in-addr.arpa \
    --name-server=ipa1.example.com.

# Add reverse zone for IPv6
ipa dnszone-add \
    1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa \
    --name-server=ipa1.example.com.

# Add PTR record
ipa dnsrecord-add 1.168.192.in-addr.arpa 21 \
    --ptr-rec=server2.example.com.

# Test PTR resolution
dig +short -x 192.168.1.21 @ipa1.example.com

# Auto-create PTR when adding A record (if zone exists)
ipa dnsrecord-add example.com server3 \
    --a-rec=192.168.1.22 \
    --a-create-reverse
```

[↑ Back to TOC](#table-of-contents)

---

## DNS Forwarders

```bash
# Add a global DNS forwarder (for all queries)
ipa dnsconfig-mod --forwarder=8.8.8.8 --forwarder=8.8.4.4

# Show current forwarder config
ipa dnsconfig-show

# Set forwarding policy
# first  = try IPA DNS, fall back to forwarder (default)
# only   = always use forwarder for external names
# none   = disable global forwarders
ipa dnsconfig-mod --forward-policy=first

# Add conditional forwarder for a specific domain
ipa dnsforwardzone-add ad.example.com \
    --forwarder=192.168.10.10 \
    --forwarder=192.168.10.11 \
    --forward-policy=only

# Show forward zone
ipa dnsforwardzone-show ad.example.com

# Find all forward zones
ipa dnsforwardzone-find

# Modify forward zone
ipa dnsforwardzone-mod ad.example.com \
    --forwarder=192.168.10.20

# Disable/enable forward zone
ipa dnsforwardzone-disable ad.example.com
ipa dnsforwardzone-enable ad.example.com

# Delete forward zone
ipa dnsforwardzone-del ad.example.com
```

[↑ Back to TOC](#table-of-contents)

---

## DNSSEC Operations

```bash
# Enable DNSSEC signing for a zone
ipa dnszone-mod example.com \
    --dnssec=true

# Check DNSSEC status
ipa dnszone-show example.com | grep -i "DNSSEC\|signing"

# List DNSSEC keys for a zone
ipa dnskey-find example.com

# Show DNSSEC key details
ipa dnskey-show example.com <key-tag>

# Check OpenDNSSEC services
sudo systemctl status ods-enforcerd ods-signerd

# Verify DNSSEC signatures
dig +dnssec ipa1.example.com @localhost
# Look for "ad" flag (authenticated data) in flags section

# Get DS record (for parent zone delegation)
dig +short example.com DS @localhost
# Paste this into parent zone (e.g., example.com) to complete chain of trust

# Manually trigger key rollover check
sudo ods-enforcer enforce

# Check DNSSEC key database
sudo ods-enforcer key list --zone=example.com --verbose
```

[↑ Back to TOC](#table-of-contents)

---

## SRV Records

```bash
# IPA auto-creates SRV records — verify they exist
# Kerberos
dig +short SRV _kerberos._tcp.example.com
dig +short SRV _kerberos._udp.example.com
dig +short SRV _kerberos-master._tcp.example.com

# LDAP
dig +short SRV _ldap._tcp.example.com
dig +short SRV _ldaps._tcp.example.com

# HTTPS/IPA API
dig +short SRV _https._tcp.example.com

# Kerberos realm TXT record
dig +short TXT _kerberos.example.com

# Manually add missing SRV record
ipa dnsrecord-add example.com _kerberos._tcp \
    --srv-rec="0 100 88 ipa1.example.com."

# SRV record format: priority weight port target.
ipa dnsrecord-add example.com _ldap._tcp \
    --srv-rec="0 100 389 ipa1.example.com." \
    --srv-rec="10 100 389 ipa2.example.com."
```

[↑ Back to TOC](#table-of-contents)

---

## DNS Diagnostics

```bash
# Test DNS resolution against IPA server
dig @ipa1.example.com ipa1.example.com A
dig @ipa1.example.com example.com SOA

# Test SRV record resolution
dig +short SRV _kerberos._tcp.EXAMPLE.COM
dig +short SRV _ldap._tcp.example.com

# Test from client
cat /etc/resolv.conf
nslookup ipa1.example.com
host ipa1.example.com

# Test reverse lookup
dig -x 192.168.1.10 @ipa1.example.com +short

# Check zone transfer (should be restricted)
dig @ipa1.example.com example.com AXFR
# Should return: "Transfer failed" if properly locked down

# Check DNS port is open
nc -zv ipa1.example.com 53
sudo ss -tlnup | grep :53

# Trace DNS resolution path
dig +trace ipa1.example.com @8.8.8.8

# Test DNSSEC validation chain
dig +dnssec +cd example.com SOA @localhost
dig +dnssec example.com SOA @localhost
```

[↑ Back to TOC](#table-of-contents)

---

## named Service

```bash
# Check named service status
sudo systemctl status named

# Reload zone data (after direct LDAP changes)
sudo systemctl reload named
# Or:
sudo rndc reload example.com

# Check named configuration syntax
sudo named-checkconf /etc/named.conf

# Check named configuration files
cat /etc/named.conf
cat /etc/named/ipa-ext.conf
cat /etc/named/ipa-options-ext.conf

# Flush DNS cache
sudo rndc flush

# Flush specific zone
sudo rndc flushname ipa1.example.com

# View named stats
sudo rndc stats
cat /var/named/data/named_stats.txt | head -50

# named logs
sudo journalctl -u named --since "1 hour ago"
sudo tail -30 /var/log/named/named.log 2>/dev/null || \
    sudo journalctl -u named -n 30

# bind-dyndb-ldap log (in named journal)
sudo journalctl -u named | grep -i "ldap\|dyndb" | tail -20
```

[↑ Back to TOC](#table-of-contents)

---

## Troubleshooting

```bash
# named won't start
sudo named-checkconf /etc/named.conf
sudo journalctl -u named --since "today" | tail -30

# DNS not resolving IPA records
# Check bind-dyndb-ldap can reach 389-DS
sudo systemctl status dirsrv@IPA-EXAMPLE-COM.service
sudo -u named ldapsearch -x -H ldap://localhost \
    -b "cn=dns,dc=ipa,dc=example,dc=com" "(idnsName=*)" idnsName | head -10

# SERVFAIL on zone
sudo rndc reload example.com
dig @localhost example.com SOA

# Record not found but was just added
sudo rndc flush
dig @localhost newrecord.example.com

# Forwarder not working
dig +short external.domain @8.8.8.8              # test forwarder directly
dig +short external.domain @ipa1.example.com  # test via IPA

# DNSSEC validation failing
# Check if zone is properly signed
dig +dnssec example.com SOA @localhost | grep -E "RRSIG|NSEC"
sudo ods-enforcer key list --zone=example.com

# Check DNSSEC healthcheck
sudo ipa-healthcheck --source ipahealthcheck.ipa.dns
```

[↑ Back to TOC](#table-of-contents)

---

*Licensed under [CC BY-NC-SA 4.0](../LICENSE.md) · © 2026 UncleJS*