# CS-02 — Kerberos Cheatsheet

> Quick-reference for Kerberos operations, ticket management, keytabs, enctypes, and troubleshooting on RHEL 10 / FreeIPA 4.12.

---

## Table of Contents

- [kinit / klist / kdestroy](#kinit--klist--kdestroy)
- [Ticket Options and Lifetimes](#ticket-options-and-lifetimes)
- [Keytab Operations](#keytab-operations)
- [Service Principals](#service-principals)
- [Enctype Reference](#enctype-reference)
- [Kerberos Policy](#kerberos-policy)
- [OTP / MFA](#otp--mfa)
- [KDC Administration (kadmin)](#kdc-administration-kadmin)
- [Cross-Realm / AD Trust Tickets](#cross-realm--ad-trust-tickets)
- [Troubleshooting](#troubleshooting)

---

## kinit / klist / kdestroy

```bash
# Basic ticket acquisition
kinit                           # uses $USER@DEFAULT_REALM
kinit admin                     # admin in default realm
kinit admin@IPA.EXAMPLE.COM     # explicit realm
kinit -l 8h admin               # ticket lifetime 8 hours
kinit -r 7d admin               # renewable lifetime 7 days
kinit -R                        # renew existing ticket

# Get forwardable/proxiable ticket
kinit -f admin                  # forwardable
kinit -p admin                  # proxiable

# Ticket from keytab (no password prompt — for automation)
kinit -kt /etc/krb5.keytab host/server.ipa.example.com
kinit -kt /etc/service.keytab -k svc@IPA.EXAMPLE.COM

# Trace kinit (debug)
KRB5_TRACE=/dev/stderr kinit admin 2>&1

# List tickets
klist                            # summary
klist -v                         # verbose (includes encryption types)
klist -e                         # show encryption types
klist -f                         # show ticket flags
klist -c /tmp/krb5cc_1000        # specific ccache
klist -l                         # list all ccaches (collection)
klist -kt /etc/krb5.keytab       # list keytab entries

# Destroy tickets
kdestroy                         # default ccache
kdestroy -A                      # all ccaches
kdestroy -c /tmp/krb5cc_1000     # specific ccache

# Get service ticket (test connectivity)
kvno host/server1.ipa.example.com
kvno -S host server1.ipa.example.com
kvno HTTP/ipa1.ipa.example.com
```

---

## Ticket Options and Lifetimes

```bash
# /etc/krb5.conf defaults
[libdefaults]
  default_realm = IPA.EXAMPLE.COM
  ticket_lifetime = 24h         # default TGT lifetime
  renew_lifetime = 7d           # renewable window
  forwardable = true
  canonicalize = false

# IPA KDC policy (managed via ipa krbtpolicy)
ipa krbtpolicy-show
ipa krbtpolicy-mod --maxlife=36000    # 10 hours in seconds
ipa krbtpolicy-mod --maxrenew=604800  # 7 days in seconds

# Per-user ticket policy override
ipa krbtpolicy-mod --user=svc_batch \
    --maxlife=3600 --maxrenew=3600

# Reset per-user policy to global
ipa krbtpolicy-reset --user=svc_batch

# Ticket flag reference
# F=Forwardable  f=forwarded  P=Proxiable  p=proxy
# D=postDateable d=postdated  R=Renewable  I=Initial
# H=hardware auth A=preauth   T=transit policy checked
```

---

## Keytab Operations

```bash
# Retrieve keytab for a service principal (run on IPA server)
ipa-getkeytab \
    -s ipa1.ipa.example.com \
    -p HTTP/webapp.ipa.example.com \
    -k /etc/httpd/conf/krb5.keytab

# Retrieve host keytab
ipa-getkeytab \
    -s ipa1.ipa.example.com \
    -p host/client1.ipa.example.com \
    -k /etc/krb5.keytab

# Retrieve with specific enctype
ipa-getkeytab \
    -s ipa1.ipa.example.com \
    -p host/client1.ipa.example.com \
    -k /etc/krb5.keytab \
    -e aes256-cts

# Add new keys without removing old ones
ipa-getkeytab \
    -s ipa1.ipa.example.com \
    -p host/client1.ipa.example.com \
    -k /etc/krb5.keytab \
    --append

# Inspect keytab contents
klist -kt /etc/krb5.keytab
ktutil -k /etc/krb5.keytab list    # alternative

# Test keytab authentication
kinit -kt /etc/krb5.keytab host/client1.ipa.example.com
klist

# Set correct permissions on keytab
sudo chown root:root /etc/krb5.keytab
sudo chmod 600 /etc/krb5.keytab

# Keytab for non-root service (e.g., HTTP)
sudo chown root:apache /etc/httpd/conf/krb5.keytab
sudo chmod 640 /etc/httpd/conf/krb5.keytab
```

---

## Service Principals

```bash
# Add a service principal
ipa service-add HTTP/webapp.ipa.example.com
ipa service-add nfs/fileserver.ipa.example.com
ipa service-add LDAP/ldapproxy.ipa.example.com

# Show principal
ipa service-show HTTP/webapp.ipa.example.com

# Add allowed kerberos enctypes restriction
ipa service-mod HTTP/webapp.ipa.example.com \
    --ipakrbauthzdata=MS-PAC

# Allow a host to retrieve the keytab
ipa service-allow-retrieve-keytab \
    HTTP/webapp.ipa.example.com \
    --hosts=webapp.ipa.example.com

# Allow a host to create the keytab (write access)
ipa service-allow-create-keytab \
    HTTP/webapp.ipa.example.com \
    --hosts=webapp.ipa.example.com

# Manage keytab retrieval delegation
ipa service-disallow-retrieve-keytab \
    HTTP/webapp.ipa.example.com \
    --hosts=oldhost.ipa.example.com

# Delete service principal
ipa service-del HTTP/webapp.ipa.example.com
```

---

## Enctype Reference

| Enctype String | Key Size | Notes |
|---------------|----------|-------|
| `aes256-cts-hmac-sha384-192` | AES-256 | **Preferred** — RFC 8009 |
| `aes256-cts-hmac-sha1-96` | AES-256 | RFC 3962 — widely supported |
| `aes128-cts-hmac-sha256-128` | AES-128 | RFC 8009 |
| `aes128-cts-hmac-sha1-96` | AES-128 | RFC 3962 |
| `des3-cbc-sha1` | 3DES | ❌ Disabled in RHEL 10 DEFAULT |
| `arcfour-hmac` (RC4) | 128-bit | ❌ Removed in RHEL 10 |
| `des-cbc-crc` / `des-cbc-md5` | DES | ❌ Removed in RHEL 10 |

```bash
# Check effective enctypes (from crypto policy)
grep -r permitted_enctypes /etc/krb5.conf /etc/krb5.conf.d/

# Restrict to AES-256 only (harden)
sudo tee /etc/krb5.conf.d/ipa-hardened-enctypes.conf << 'EOF'
[libdefaults]
 default_tkt_enctypes = aes256-cts-hmac-sha384-192 aes256-cts-hmac-sha1-96
 default_tgs_enctypes = aes256-cts-hmac-sha384-192 aes256-cts-hmac-sha1-96
 permitted_enctypes   = aes256-cts-hmac-sha384-192 aes256-cts-hmac-sha1-96
 allow_weak_crypto    = false
EOF
```

---

## Kerberos Policy

```bash
# Global ticket policy
ipa krbtpolicy-show

# Modify global policy
ipa krbtpolicy-mod \
    --maxlife=36000 \
    --maxrenew=604800
# maxlife=36000 = 10 hours; maxrenew=604800 = 7 days

# Per-user policy
ipa krbtpolicy-mod --user=admin --maxlife=7200

# Reset per-user to global
ipa krbtpolicy-reset --user=admin

# Password policy (affects Kerberos key generation)
ipa pwpolicy-show global_policy
ipa pwpolicy-mod global_policy \
    --minlength=16 --maxfail=5 --lockouttime=600

# Unlock locked account (too many failed kinit attempts)
ipa user-unlock jsmith

# Check account lockout status
ipa user-show jsmith | grep -i "Kerberos last\|locked\|disabled"
```

---

## OTP / MFA

```bash
# Add OTP token for user (TOTP — Google Authenticator compatible)
ipa otptoken-add \
    --type=totp \
    --owner=jsmith \
    --desc="jsmith phone"

# The command returns a QR code URI — scan with authenticator app

# Show token
ipa otptoken-show <token-id>

# List tokens for user
ipa otptoken-find --owner=jsmith

# Disable token
ipa otptoken-disable <token-id>

# Delete token
ipa otptoken-del <token-id>

# Enable OTP auth for user
ipa user-mod jsmith --user-auth-type=otp

# Enable OTP + password (two-factor)
ipa user-mod jsmith \
    --user-auth-type=password \
    --user-auth-type=otp

# Enforce OTP globally for admins group
ipa group-mod admins --setattr=ipauserauthtype=otp

# Test OTP login (enter password immediately followed by OTP code, no space)
kinit jsmith
# Password: MyPassword123456789  ← password+OTP concatenated

# RADIUS proxy for hardware tokens
ipa radiusproxy-add corp-radius \
    --server=radius.example.com \
    --secret='RadiusSecret!'
ipa user-mod jsmith --radius=corp-radius
```

---

## KDC Administration (kadmin)

```bash
# Local kadmin (must be root on KDC host)
sudo kadmin.local

# Remote kadmin (requires Kerberos auth)
kadmin -p admin/admin@IPA.EXAMPLE.COM

# kadmin commands:
# List principals
kadmin.local -q "listprincs"
kadmin.local -q "listprincs | grep admin"

# Get principal details
kadmin.local -q "getprinc admin"
kadmin.local -q "getprinc host/server1.ipa.example.com"

# Change principal flags
kadmin.local -q "modprinc +requires_preauth admin"
kadmin.local -q "modprinc -allow_forwardable svc_noproxy"

# Key operations (low-level — prefer ipa-getkeytab)
kadmin.local -q "ktadd -k /tmp/service.keytab HTTP/webapp.ipa.example.com"
kadmin.local -q "ktremove -k /tmp/service.keytab HTTP/webapp.ipa.example.com all"

# Check database
sudo kdb5_util dump /tmp/kdb-export.txt

# Verify stash file
sudo kdb5_util stash -P 'MasterPassword' 2>/dev/null || \
    echo "Stash file exists"
```

---

## Cross-Realm / AD Trust Tickets

```bash
# Kinit as an AD user (requires AD trust)
kinit aduser@AD.EXAMPLE.COM
klist

# Get cross-realm TGT manually
kinit -f aduser@AD.EXAMPLE.COM     # forwardable required for delegation

# Test cross-realm service ticket
kvno host/ipa-client.ipa.example.com@IPA.EXAMPLE.COM

# Verify cross-realm krbtgt principal exists on IPA
sudo kadmin.local -q "getprinc krbtgt/AD.EXAMPLE.COM@IPA.EXAMPLE.COM"
sudo kadmin.local -q "getprinc krbtgt/IPA.EXAMPLE.COM@AD.EXAMPLE.COM"

# Check PAC in ticket (requires LDAP PAC inspection — use SSSD logs)
id aduser@ad.example.com
```

---

## Troubleshooting

```bash
# Clock skew error
timedatectl status
sudo chronyc makestep
kinit admin

# Cannot contact KDC
dig +short SRV _kerberos._tcp.IPA.EXAMPLE.COM
nc -zv ipa1.ipa.example.com 88

# Client not found
ipa user-show USERNAME    # verify user exists
kinit USERNAME@IPA.EXAMPLE.COM    # explicit realm

# Preauthentication failed
ipa user-unlock USERNAME
ipa user-show USERNAME | grep -i "Kerberos last failed auth"

# Enctype not supported
klist -e    # check what enctype was used
grep permitted_enctypes /etc/krb5.conf /etc/krb5.conf.d/*

# KDC logs
sudo tail -50 /var/log/krb5kdc.log
sudo journalctl -u krb5kdc --since "1 hour ago"

# Trace full Kerberos exchange
KRB5_TRACE=/dev/stderr kinit admin 2>&1 | grep -v "^$"

# Check KDC is running
sudo systemctl status krb5kdc kadmin
```
