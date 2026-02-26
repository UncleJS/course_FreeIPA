# CS-07 — AD Trust Cheatsheet

> Quick-reference for Active Directory trust setup, management, ID mapping, and troubleshooting in FreeIPA on RHEL 10.

---

## Table of Contents

- [Trust Setup](#trust-setup)
- [Trust Management](#trust-management)
- [ID Ranges and Mapping](#id-ranges-and-mapping)
- [External Groups](#external-groups)
- [HBAC and Sudo for AD Users](#hbac-and-sudo-for-ad-users)
- [User Overrides and ID Views](#user-overrides-and-id-views)
- [Winbind and SSSD Diagnostics](#winbind-and-sssd-diagnostics)
- [Troubleshooting](#troubleshooting)

---

## Trust Setup

```bash
# Step 1: Install AD trust packages (on every IPA trust controller)
sudo dnf install -y ipa-server-trust-ad

# Step 2: Open firewall ports for AD trust
sudo firewall-cmd --permanent --add-service=freeipa-trust
sudo firewall-cmd --reload

# Step 3: Add DNS conditional forwarder for AD domain
kinit admin
ipa dnsforwardzone-add ad.example.com \
    --forwarder=192.168.10.10 \
    --forwarder=192.168.10.11 \
    --forward-policy=only

# Verify DNS resolution of AD services
dig +short SRV _ldap._tcp.ad.example.com
dig +short SRV _kerberos._tcp.ad.example.com
dig +short A dc1.ad.example.com

# Step 4: Run ipa-adtrust-install on each IPA trust controller
sudo ipa-adtrust-install \
    --netbios-name=IPA \
    --add-sids \
    --add-agents \
    --admin-password='AdminPassword123!'

# Verify Samba/winbind services are up
sudo systemctl status smb winbind
sudo testparm -s | grep workgroup
ipa dnszone-show _msdcs.ipa.example.com

# Step 5: Create the trust
ipa trust-add ad.example.com \
    --admin=Administrator \
    --password \
    --type=ad
```

---

## Trust Management

```bash
# List all trusts
ipa trust-find

# Show trust details
ipa trust-show ad.example.com

# Fetch sub-domains from trusted forest
ipa trust-fetch-domains ad.example.com

# Modify trust
ipa trust-mod ad.example.com \
    --two-way=true    # upgrade to two-way trust

# Delete trust
ipa trust-del ad.example.com

# Verify trust from Samba
sudo wbinfo --ping-dc --domain=ad.example.com
sudo wbinfo --check-secret --domain=ad.example.com
sudo wbinfo --trusted-domains

# List AD users (use with caution in large domains)
sudo wbinfo -u --domain=ad.example.com | head -20
sudo wbinfo -g --domain=ad.example.com | head -20

# Look up AD user SID
sudo wbinfo --name-to-sid='AD\aduser1'

# Look up SID to name
sudo wbinfo --sid-to-name='S-1-5-21-...-1105'
```

---

## ID Ranges and Mapping

```bash
# List all ID ranges
ipa idrange-find

# Show specific range
ipa idrange-show 'AD.EXAMPLE.COM_id_range'

# Range type options:
# ipa-ad-trust        = RID-based (default, no POSIX attrs in AD)
# ipa-ad-trust-posix  = Read uidNumber/gidNumber from AD

# Modify range base ID (to avoid conflicts with IPA range)
ipa idrange-mod 'AD.EXAMPLE.COM_id_range' \
    --base-id=2000000

# Change range type to POSIX (if AD has Unix Attributes schema)
ipa trust-mod ad.example.com \
    --range-type=ipa-ad-trust-posix

# How UID is calculated (RID-based):
# UID = BaseID + RID
# Example: BaseID=1000000, RID=1105 → UID=1001105

# Verify ID resolution
id aduser1@ad.example.com
getent passwd aduser1@ad.example.com

# Flush SSSD cache to pick up range changes
sudo sss_cache -E
sudo systemctl restart sssd
```

---

## External Groups

```bash
# Create an external group (references AD group by SID or name)
ipa group-add ext_linux_admins \
    --desc="External wrapper for AD linux_admins" \
    --external

# Add AD group to external group by name
ipa group-add-member ext_linux_admins \
    --external='AD\linux_admins'

# Add AD group by SID
ipa group-add-member ext_linux_admins \
    --external='S-1-5-21-1234567890-...-1234'

# Show external group members (SIDs)
ipa group-show ext_linux_admins --all | grep member

# Create POSIX group and nest external group in it
ipa group-add ipa_linux_admins \
    --desc="POSIX group for AD linux admins"
ipa group-add-member ipa_linux_admins \
    --groups=ext_linux_admins

# Verify group membership for AD user
id aduser1@ad.example.com
getent group ipa_linux_admins

# Remove AD group from external group
ipa group-remove-member ext_linux_admins \
    --external='AD\linux_admins'

# List all external groups
ipa group-find --external=true
```

---

## HBAC and Sudo for AD Users

```bash
# AD users CANNOT be added directly to HBAC/sudo rules
# They must go through: AD user → AD group → external group → IPA POSIX group → rule

# Full HBAC setup for AD users:
# 1. Create external group
ipa group-add ext_ad_ssh --external
ipa group-add-member ext_ad_ssh --external='AD\linux_users'

# 2. Create POSIX group
ipa group-add ipa_ad_ssh
ipa group-add-member ipa_ad_ssh --groups=ext_ad_ssh

# 3. Create HBAC rule
ipa hbacrule-add allow_ad_users_ssh \
    --servicecat=all
ipa hbacrule-add-user allow_ad_users_ssh \
    --groups=ipa_ad_ssh
ipa hbacrule-add-host allow_ad_users_ssh \
    --hostgroups=managed_hosts

# 4. Test HBAC
ipa hbactest \
    --user=aduser1@ad.example.com \
    --host=server1.ipa.example.com \
    --service=sshd

# Full sudo setup for AD users (same group chain):
ipa group-add ext_ad_sudo --external
ipa group-add-member ext_ad_sudo --external='AD\sudo_users'
ipa group-add ipa_ad_sudo
ipa group-add-member ipa_ad_sudo --groups=ext_ad_sudo

ipa sudorule-add ad_sudo_all
ipa sudorule-add-user ad_sudo_all --groups=ipa_ad_sudo
ipa sudorule-add-runasuser ad_sudo_all --users=root
ipa sudorule-add-option ad_sudo_all --sudooption='!authenticate'
```

---

## User Overrides and ID Views

```bash
# Create user override in Default Trust View
ipa idoverrideuser-add 'Default Trust View' \
    aduser1@ad.example.com \
    --login=aduser1 \
    --homedir=/home/aduser1 \
    --shell=/bin/bash

# Add SSH key via override
ipa idoverrideuser-mod 'Default Trust View' \
    aduser1@ad.example.com \
    --sshpubkey="ssh-ed25519 AAAA..."

# Show override
ipa idoverrideuser-show 'Default Trust View' \
    aduser1@ad.example.com

# Create custom ID view
ipa idview-add RestrictedView \
    --desc="Restricted shell for DMZ hosts"

# Apply view to specific hosts
ipa idview-apply RestrictedView \
    --hosts=dmz1.ipa.example.com

# Add override in custom view
ipa idoverrideuser-add RestrictedView \
    aduser1@ad.example.com \
    --shell=/bin/sh \
    --homedir=/tmp/aduser1

# List all ID views
ipa idview-find

# Remove view from host
ipa idview-unapply RestrictedView \
    --hosts=dmz1.ipa.example.com
```

---

## Winbind and SSSD Diagnostics

```bash
# Winbind connectivity tests
sudo wbinfo --ping-dc --domain=ad.example.com
sudo wbinfo --check-secret --domain=ad.example.com
sudo wbinfo -u --domain=ad.example.com | head -10
sudo wbinfo -g --domain=ad.example.com | head -10
sudo wbinfo --name-to-sid='AD\aduser1'
sudo wbinfo --sid-to-uid='S-1-5-21-...-1105'

# Samba configuration check
sudo testparm -s

# Samba logs
sudo journalctl -u smb -u winbind --since "30 min ago"
sudo tail -50 /var/log/samba/log.wb-AD

# SSSD AD subdomain status
sudo sssctl domain-status ad.example.com

# SSSD user lookup
id aduser1@ad.example.com
getent passwd aduser1@ad.example.com
getent group 'domain users@ad.example.com'

# Flush SSSD cache for AD domain
sudo sss_cache -E -d ad.example.com
sudo sss_cache -u aduser1@ad.example.com

# Enable SSSD debug for AD
sudo sss_debuglevel --domain=ad.example.com 9
sudo systemctl restart sssd
sudo tail -f /var/log/sssd/sssd_ad.example.com.log
```

---

## Troubleshooting

```bash
# Cannot create trust
# Check DNS
dig +short SRV _ldap._tcp.ad.example.com
dig +short SRV _kerberos._tcp.ad.example.com
# Check network
nc -zv dc1.ad.example.com 445
nc -zv dc1.ad.example.com 88

# AD user not found
sudo sss_cache -E
sudo systemctl restart sssd
id aduser1@ad.example.com
# If still not found: check wbinfo -u --domain=ad.example.com

# UID returns 4294967295 (NXUNKNOWN)
# ID range mismatch or SSSD cache stale
ipa idrange-show 'AD.EXAMPLE.COM_id_range'
sudo sss_cache -E
sudo systemctl restart sssd

# Trust breaks after DC change in AD
ipa trust-fetch-domains ad.example.com
sudo wbinfo --ping-dc --domain=ad.example.com

# Access denied after login
ipa hbactest --user=aduser1@ad.example.com \
    --host=server1.ipa.example.com --service=sshd
# Check external group chain:
sudo wbinfo --name-to-sid='AD\linux_users'
ipa group-show ext_ad_ssh --all | grep member

# Clock skew
timedatectl status
sudo chronyc makestep

# RC4/weak enctypes: AD doesn't support AES
# Fix on AD DC (PowerShell):
# Set-ADUser aduser1 -KerberosEncryptionType AES256,AES128
```
