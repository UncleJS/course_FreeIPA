# CS-05 — RBAC and Delegation Cheatsheet
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](../LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![FreeIPA](https://img.shields.io/badge/FreeIPA-v4.12-blue)](https://www.freeipa.org)

> Quick-reference for roles, privileges, permissions, ACI delegation, and RBAC operations in FreeIPA on RHEL 10.

---

> 🔁 **See also:** [Module 08 — RBAC & Delegation](../08_rbac_delegation.md)

---

## Table of Contents

- [Roles](#roles)
- [Privileges](#privileges)
- [Permissions](#permissions)
- [Self-Service Permissions](#self-service-permissions)
- [Delegation (User-to-User)](#delegation-user-to-user)
- [Built-in Roles Reference](#built-in-roles-reference)
- [ACI Inspection](#aci-inspection)
- [Troubleshooting Access Issues](#troubleshooting-access-issues)

---

## Roles

```bash
# List all roles
ipa role-find

# Show role details (members and privileges)
ipa role-show "User Administrator"
ipa role-show "Helpdesk"

# Create a custom role
ipa role-add "App Team Admin" \
    --desc="Admin role for application team"

# Add users/groups to a role
ipa role-add-member "App Team Admin" \
    --users=jsmith
ipa role-add-member "App Team Admin" \
    --groups=app_admins
ipa role-add-member "App Team Admin" \
    --hosts=mgmt1.example.com

# Remove members from role
ipa role-remove-member "App Team Admin" \
    --users=jsmith

# Add privileges to a role
ipa role-add-privilege "App Team Admin" \
    --privileges="User Administrators"
ipa role-add-privilege "App Team Admin" \
    --privileges="Group Administrators"

# Remove privilege from role
ipa role-remove-privilege "App Team Admin" \
    --privileges="User Administrators"

# Delete role
ipa role-del "App Team Admin"

# Check which roles a user has (indirect — via group membership)
ipa user-show jsmith --all | grep "memberof"
```

[↑ Back to TOC](#table-of-contents)

---

## Privileges

```bash
# List all privileges
ipa privilege-find

# Show privilege details (which permissions it grants)
ipa privilege-show "User Administrators"

# Create a custom privilege
ipa privilege-add "Manage App Servers" \
    --desc="Can manage app server hosts and services"

# Add permissions to a privilege
ipa privilege-add-permission "Manage App Servers" \
    --permissions="System: Add Hosts" \
    --permissions="System: Modify Hosts" \
    --permissions="System: Manage Services"

# Remove permission from privilege
ipa privilege-remove-permission "Manage App Servers" \
    --permissions="System: Add Hosts"

# Delete privilege
ipa privilege-del "Manage App Servers"
```

[↑ Back to TOC](#table-of-contents)

---

## Permissions

```bash
# List all permissions
ipa permission-find

# List permissions matching keyword
ipa permission-find user
ipa permission-find host

# Show permission details (what it allows and in which subtree)
ipa permission-show "System: Add Users"
ipa permission-show "System: Modify Users"

# Create a custom permission
ipa permission-add "Allow read email addresses" \
    --type=user \
    --attrs=mail \
    --right=read \
    --bindtype=permission

# Permission bindtypes:
# permission  = granted via privilege/role chain
# all         = granted to all authenticated users
# anonymous   = granted to unauthenticated (anonymous) queries
# self        = user can apply to themselves (see self-service)

# Modify permission (add/change attributes)
ipa permission-mod "Allow read email addresses" \
    --attrs=mail --attrs=telephoneNumber

# Delete custom permission
ipa permission-del "Allow read email addresses"

# Find which role/privilege grants a permission
ipa permission-show "System: Add Users" | grep "Granted to"
```

[↑ Back to TOC](#table-of-contents)

---

## Self-Service Permissions

```bash
# List self-service permissions
ipa selfservice-find

# Show a self-service permission
ipa selfservice-show "Users can manage their own name details"

# Create a self-service permission (users modify own attributes)
ipa selfservice-add "Users can manage their own phone" \
    --attrs=telephoneNumber \
    --attrs=mobile

# Modify self-service permission
ipa selfservice-mod "Users can manage their own phone" \
    --attrs=telephoneNumber \
    --attrs=mobile \
    --attrs=homePhone

# Delete self-service permission
ipa selfservice-del "Users can manage their own phone"
```

[↑ Back to TOC](#table-of-contents)

---

## Delegation (User-to-User)

```bash
# List delegation rules
ipa delegation-find

# Show delegation rule
ipa delegation-show "helpdesk user managers"

# Create delegation rule
# (group A members can modify attributes X of group B members)
ipa delegation-add "helpdesk manages passwords" \
    --attrs=userpassword \
    --group=helpdesk \
    --memberof=all_users

# Another example: IT team can manage phone numbers for employees
ipa delegation-add "it manages contacts" \
    --attrs=telephoneNumber \
    --attrs=mobile \
    --group=it_team \
    --memberof=employees

# Modify delegation rule
ipa delegation-mod "helpdesk manages passwords" \
    --attrs=userpassword \
    --attrs=krbPasswordExpiration

# Delete delegation rule
ipa delegation-del "helpdesk manages passwords"
```

[↑ Back to TOC](#table-of-contents)

---

## Built-in Roles Reference

| Role | Description | Typical Use |
|------|-------------|-------------|
| `admin` | Full IPA admin (superuser) | IPA system administrators only |
| `User Administrator` | Add/modify/delete users and groups | HR / IT user provisioning |
| `Group Administrator` | Manage groups | Team leads |
| `Host Administrator` | Add/modify/delete hosts | Infrastructure team |
| `Host Enrollment` | Can enroll new hosts | Automation / provisioning accounts |
| `Helpdesk` | Reset passwords, unlock accounts | IT helpdesk |
| `Service Administrator` | Manage service principals | Application teams |
| `DNS Administrator` | Manage DNS zones and records | DNS/network team |
| `IT Security Specialist` | Manage HBAC, sudo, password policy | Security team |
| `CA Administrator` | Manage certificates and profiles | PKI team |

```bash
# Assign a user to built-in Helpdesk role
ipa role-add-member "Helpdesk" --users=helpdesk_user

# Assign a group to User Administrator role
ipa role-add-member "User Administrator" \
    --groups=user_provisioning_team

# Show all members of a built-in role
ipa role-show "Helpdesk"
```

[↑ Back to TOC](#table-of-contents)

---

## ACI Inspection

```bash
# View raw ACIs in LDAP (for advanced debugging)
sudo ldapsearch -x -H ldaps://ipa1.example.com \
    -D "uid=admin,cn=users,cn=accounts,dc=ipa,dc=example,dc=com" \
    -W \
    -b "dc=ipa,dc=example,dc=com" \
    -s one \
    "(objectClass=*)" \
    aci | grep "^aci:" | head -20

# ACIs on user subtree
sudo ldapsearch -x -H ldap://localhost \
    -D "cn=Directory Manager" -W \
    -b "cn=users,cn=accounts,dc=ipa,dc=example,dc=com" \
    -s base "(objectClass=*)" aci

# ACIs on a specific user
sudo ldapsearch -x -H ldap://localhost \
    -D "cn=Directory Manager" -W \
    -b "uid=jsmith,cn=users,cn=accounts,dc=ipa,dc=example,dc=com" \
    -s base "(objectClass=*)" aci

# IPA manages ACIs automatically via permissions
# Do NOT manually edit ACIs — use ipa permission-* instead
```

[↑ Back to TOC](#table-of-contents)

---

## Troubleshooting Access Issues

```bash
# If the problem is host login or sudo access, check policy order first:
# 1. HBAC decides whether the user gets a session on the host at all.
# 2. Sudo rules apply only after login succeeds.

# Login gate first
ipa hbactest --user=jsmith --host=server1.example.com --service=sshd

# Then inspect sudo policy from the client host
sudo -l -U jsmith

# User can't perform action — determine what permissions are needed

# 1. Check what roles user is in
ipa user-show jsmith --all | grep -i "memberof.*role"

# 2. Check what privileges those roles grant
ipa role-show "User Administrator" | grep "Privileges"

# 3. Check what permissions those privileges include
ipa privilege-show "User Administrators" | grep "Permissions"

# 4. Verify the permission covers the required attribute/operation
ipa permission-show "System: Modify Users" | grep -E "attrs|rights|subtree"

# 5. Test with ipa --debug (shows LDAP operations)
ipa --debug user-mod jsmith --title="Test" 2>&1 | \
    grep -E "RESULT|error|denied"

# 6. Check 389-DS access log for the actual LDAP result code
sudo tail -20 /var/log/dirsrv/slapd-IPA-EXAMPLE-COM/access | \
    grep -E "ERR=50|RESULT err=50"
# ERR=50 = Insufficient access rights

# 7. Add required permission/privilege to user's role
ipa role-add-privilege "Custom Role" \
    --privileges="System: Modify Users"
```

[↑ Back to TOC](#table-of-contents)

---

*Licensed under [CC BY-NC-SA 4.0](../LICENSE.md) · © 2026 UncleJS*
