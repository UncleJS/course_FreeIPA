# CS-01 — Daily Administration Cheatsheet

> Quick-reference commands for day-to-day FreeIPA operations on RHEL 10.

---

## Table of Contents

- [Service Management](#service-management)
- [User Operations](#user-operations)
- [Group Operations](#group-operations)
- [Host Operations](#host-operations)
- [HBAC Operations](#hbac-operations)
- [Sudo Rule Operations](#sudo-rule-operations)
- [Password Policy](#password-policy)
- [DNS Operations](#dns-operations)
- [Kerberos Tickets](#kerberos-tickets)
- [Search and Find](#search-and-find)
- [Bulk Operations](#bulk-operations)

---

## Service Management

```bash
# Start / stop / restart all IPA services (correct order)
sudo ipactl start
sudo ipactl stop
sudo ipactl restart
sudo ipactl status

# Individual services
sudo systemctl status dirsrv@IPA-EXAMPLE-COM.service
sudo systemctl restart krb5kdc kadmin httpd
sudo systemctl restart pki-tomcatd@pki-tomcat.service

# Run health check
sudo ipa-healthcheck --all

# Check version
ipa --version
rpm -q freeipa-server
```

---

## User Operations

```bash
# Add user
ipa user-add jsmith \
    --first=John --last=Smith \
    --email=jsmith@example.com \
    --shell=/bin/bash \
    --homedir=/home/jsmith

# Show user
ipa user-show jsmith
ipa user-show jsmith --all    # include all attributes

# Modify user
ipa user-mod jsmith --title="Senior Engineer"
ipa user-mod jsmith --shell=/bin/zsh
ipa user-mod jsmith --email=j.smith@example.com

# Set / reset password
ipa passwd jsmith                             # interactive
echo 'NewPass123!' | ipa passwd jsmith       # non-interactive

# Disable / enable account
ipa user-disable jsmith
ipa user-enable jsmith

# Unlock locked account
ipa user-unlock jsmith

# Delete user (archive — never hard deletes)
ipa user-del jsmith

# Preserve user on delete (can be undelete later)
ipa user-del jsmith --preserve

# Undelete preserved user
ipa user-undel jsmith

# Set authentication method (password / OTP / password+OTP)
ipa user-mod jsmith --user-auth-type=otp
ipa user-mod jsmith --user-auth-type=password --user-auth-type=otp

# Add SSH public key
ipa user-mod jsmith --sshpubkey="ssh-ed25519 AAAA... jsmith@host"

# Force password reset at next login
ipa user-mod jsmith \
    --setattr=krbPasswordExpiration=19700101000000Z
```

---

## Group Operations

```bash
# Add group (POSIX)
ipa group-add developers --desc="Development team"

# Add group (non-POSIX / external)
ipa group-add ext_ad_users --external

# Add members to group
ipa group-add-member developers --users=jsmith --users=bwayne
ipa group-add-member developers --groups=contractors

# Remove members
ipa group-remove-member developers --users=jsmith

# Show group
ipa group-show developers
ipa group-show developers --all

# Find groups
ipa group-find
ipa group-find --user=jsmith    # groups jsmith belongs to

# Modify group
ipa group-mod developers --desc="Dev and Ops team"

# Delete group
ipa group-del developers
```

---

## Host Operations

```bash
# Add host
ipa host-add server1.ipa.example.com \
    --ip-address=192.168.1.20 \
    --desc="Production web server"

# Show host
ipa host-show server1.ipa.example.com

# Modify host
ipa host-mod server1.ipa.example.com --desc="Prod web server v2"

# Add SSH public key to host
ipa host-mod server1.ipa.example.com \
    --sshpubkey="ecdsa-sha2-nistp256 AAAA..."

# Add host to host group
ipa hostgroup-add webservers --desc="Web server hosts"
ipa hostgroup-add-member webservers \
    --hosts=server1.ipa.example.com

# Enroll client (run on client host)
sudo ipa-client-install \
    --server=ipa1.ipa.example.com \
    --domain=ipa.example.com \
    --realm=IPA.EXAMPLE.COM \
    --principal=admin \
    --password='AdminPassword123!' \
    --mkhomedir

# Unenroll client
sudo ipa-client-install --uninstall

# Get host keytab
ipa-getkeytab -s ipa1.ipa.example.com \
    -p host/server1.ipa.example.com \
    -k /etc/krb5.keytab

# Delete host
ipa host-del server1.ipa.example.com
```

---

## HBAC Operations

```bash
# Disable the allow-all rule (IMPORTANT: do this before adding specific rules)
ipa hbacrule-disable allow_all

# Add HBAC rule
ipa hbacrule-add allow_ssh_devs \
    --desc="Allow devs to SSH to dev hosts" \
    --servicecat=all

# Add users/groups to rule
ipa hbacrule-add-user allow_ssh_devs --groups=developers

# Add hosts/hostgroups to rule
ipa hbacrule-add-host allow_ssh_devs --hostgroups=devservers

# Add specific services
ipa hbacrule-add-service allow_ssh_devs --hbacsvcs=sshd

# Show rule
ipa hbacrule-show allow_ssh_devs

# Test access
ipa hbactest \
    --user=jsmith \
    --host=devserver1.ipa.example.com \
    --service=sshd

# Disable/enable rule
ipa hbacrule-disable allow_ssh_devs
ipa hbacrule-enable allow_ssh_devs

# Delete rule
ipa hbacrule-del allow_ssh_devs
```

---

## Sudo Rule Operations

```bash
# Add sudo rule
ipa sudorule-add allow_restart_web \
    --desc="Allow devs to restart web services"

# Add users/groups
ipa sudorule-add-user allow_restart_web --groups=developers

# Add hosts
ipa sudorule-add-host allow_restart_web --hostgroups=webservers

# Add commands
ipa sudocmd-add /usr/bin/systemctl
ipa sudorule-add-allow-command allow_restart_web \
    --sudocmds=/usr/bin/systemctl

# Add run-as user
ipa sudorule-add-runasuser allow_restart_web --users=root

# Add sudo option (e.g., no password required)
ipa sudorule-add-option allow_restart_web \
    --sudooption='!authenticate'

# Show rule
ipa sudorule-show allow_restart_web

# Test on client
sudo -l -U jsmith

# Delete rule
ipa sudorule-del allow_restart_web
```

---

## Password Policy

```bash
# Show global policy
ipa pwpolicy-show global_policy

# Modify global policy
ipa pwpolicy-mod global_policy \
    --minlength=16 \
    --minclasses=3 \
    --history=12 \
    --maxlife=90 \
    --maxfail=5 \
    --failinterval=120 \
    --lockouttime=600

# Create per-group policy
ipa pwpolicy-add admins_policy \
    --group=admins \
    --minlength=20 \
    --maxlife=60 \
    --priority=20

# Show all policies
ipa pwpolicy-find

# Simulate policy for a user
ipa user-show jsmith | grep "Kerberos password expiration"
```

---

## DNS Operations

```bash
# Add A record
ipa dnsrecord-add ipa.example.com server2 \
    --a-rec=192.168.1.21

# Add CNAME record
ipa dnsrecord-add ipa.example.com www \
    --cname-rec=webserver.ipa.example.com.

# Add PTR record
ipa dnsrecord-add 1.168.192.in-addr.arpa 21 \
    --ptr-rec=server2.ipa.example.com.

# Add MX record
ipa dnsrecord-add ipa.example.com @ \
    --mx-rec="10 mail.ipa.example.com."

# Delete record
ipa dnsrecord-del ipa.example.com server2 \
    --a-rec=192.168.1.21

# Show zone
ipa dnszone-show ipa.example.com

# Find records
ipa dnsrecord-find ipa.example.com
ipa dnsrecord-find ipa.example.com --name=server2

# Add conditional forwarder
ipa dnsforwardzone-add external.com \
    --forwarder=8.8.8.8 \
    --forward-policy=only
```

---

## Kerberos Tickets

```bash
# Get ticket
kinit admin
kinit jsmith@IPA.EXAMPLE.COM

# Get ticket with OTP
kinit jsmith    # enter password+OTP when prompted

# List tickets
klist
klist -v        # verbose with encryption details

# Renew ticket
kinit -R

# Destroy ticket
kdestroy
kdestroy -A     # destroy all ccaches

# Test specific service ticket
kvno host/server1.ipa.example.com

# Check ticket with trace
KRB5_TRACE=/dev/stderr kinit jsmith 2>&1 | head -30

# Get ticket from keytab (for scripts/automation)
kinit -kt /etc/krb5.keytab host/server1.ipa.example.com
```

---

## Search and Find

```bash
# Find users matching criteria
ipa user-find --last=Smith
ipa user-find --email='@example.com'
ipa user-find --disabled=true
ipa user-find --sizelimit=500    # increase result limit

# Find all objects of a type
ipa user-find --all --sizelimit=0
ipa host-find --sizelimit=0
ipa group-find --sizelimit=0

# Find hosts by OS or description
ipa host-find --desc="web"

# Find certificates
ipa cert-find --subject=jsmith
ipa cert-find --revocation-reason=0   # find unrevoked certs
ipa cert-find --validnotafter-from=20260101000000Z

# Find services
ipa service-find host/server1.ipa.example.com
ipa service-find --sizelimit=0 | grep "Principal"
```

---

## Bulk Operations

```bash
# Bulk add users from CSV (bash loop)
while IFS=',' read login first last email; do
    ipa user-add "$login" \
        --first="$first" \
        --last="$last" \
        --email="$email" \
        --shell=/bin/bash && echo "Added: $login"
done < users.csv

# Bulk add to group
for user in jsmith bwayne pparker; do
    ipa group-add-member developers --users="$user"
done

# Export user list to CSV
ipa user-find --all --raw --json --sizelimit=0 | \
    python3 -c "
import json,sys
data=json.load(sys.stdin)
for u in data['result']['result']:
    uid = u.get('uid',[''])[0]
    first = u.get('givenname',[''])[0]
    last = u.get('sn',[''])[0]
    email = u.get('mail',[''])[0]
    print(f'{uid},{first},{last},{email}')
"
```
