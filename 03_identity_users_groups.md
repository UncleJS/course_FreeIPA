# Module 03 — Identity: Users, Groups, and ID Management
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](./LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![FreeIPA](https://img.shields.io/badge/FreeIPA-v4.12-blue)](https://www.freeipa.org)

> Managing users, groups, automember rules, ID ranges, and the user lifecycle
> in FreeIPA. All operations use the `ipa` CLI or Web UI.

## Table of Contents

- [Recommended Background](#recommended-background)
- [Learning Outcomes](#learning-outcomes)
- [1. User Management](#1-user-management)
  - [1.1 User Attributes](#11-user-attributes)
  - [1.2 Creating Users](#12-creating-users)
  - [1.3 Modifying Users](#13-modifying-users)
  - [1.4 User Lifecycle](#14-user-lifecycle)
- [2. Group Management](#2-group-management)
  - [2.1 Group Types](#21-group-types)
  - [2.2 Creating and Managing Groups](#22-creating-and-managing-groups)
  - [2.3 Nested Groups](#23-nested-groups)
- [3. Automember Rules](#3-automember-rules)
  - [3.1 How Automember Works](#31-how-automember-works)
  - [3.2 Configuring Automember Rules](#32-configuring-automember-rules)
- [4. ID Ranges](#4-id-ranges)
  - [4.1 DNA ID Ranges](#41-dna-id-ranges)
  - [4.2 Trust ID Ranges](#42-trust-id-ranges)
- [5. ID Views](#5-id-views)
- [6. Staged Users and User Lifecycle](#6-staged-users-and-user-lifecycle)
- [7. Password Policies](#7-password-policies)
- [8. Lab — User and Group Operations](#8-lab--user-and-group-operations)
- [Key Takeaways](#key-takeaways)


---

## Recommended Background

- Complete Modules 00 through 02.
- Basic understanding of POSIX users, groups, and UID/GID concepts.
- An installed and healthy IPA server ready for identity data.

## Learning Outcomes

By the end of this module, you should be able to:

- Create and manage users, groups, and group membership.
- Explain automember rules and ID range behavior.
- Apply consistent naming and lifecycle practices to identity objects.
- Prepare accounts needed for client and policy labs.

---

## 1. User Management

### 1.1 User Attributes

Every IPA user is an LDAP object with both standard POSIX attributes and
Kerberos principal attributes:

| Attribute | LDAP attr | Notes |
|-----------|-----------|-------|
| Username | `uid` | Login name, must be unique |
| UID | `uidNumber` | POSIX UID, assigned from DNA range |
| GID | `gidNumber` | Primary group GID |
| Full name | `cn` | Display name |
| First/last | `givenName`, `sn` | Required |
| Email | `mail` | Used for notifications |
| Shell | `loginShell` | Default `/bin/bash` |
| Home dir | `homeDirectory` | `/home/username` |
| Kerberos principal | `krbPrincipalName` | `uid@REALM` |
| Password | (krb5 key) | Stored as Kerberos keys, not plaintext |
| Account expiry | `krbPrincipalExpiration` | Whole account stops working after this date |
| Password expiry | `krbPasswordExpiration` | When the password must be changed (set by password policy) |
| Account disabled | `nsAccountLock` | `TRUE` = locked |

### 1.2 Creating Users

```bash
# Minimal user creation
ipa user-add jdoe --first=John --last=Doe

# Full user creation
ipa user-add jdoe \
  --first=John \
  --last=Doe \
  --email=jdoe@example.com \
  --phone="+1 555-0100" \
  --title="Senior Engineer" \
  --shell=/bin/bash \
  --homedir=/home/jdoe \
  --password   # prompts for initial password

# Create user and set password non-interactively (CI/CD)
# --password prompts TWICE (set + verify), so feed the value twice via printf.
# A single piped line fails the verification prompt.
P='TempP@ss2026!'
printf '%s\n%s\n' "$P" "$P" | ipa user-add jdoe \
  --first=John --last=Doe --password

# Alternative: let IPA generate a random initial password (printed in output)
ipa user-add jdoe --first=John --last=Doe --random
```

> 📝 New users are forced to change their password on first login. This is
> controlled by the password policy. The initial password set by admin is
> a "temporary" credential.

### 1.3 Modifying Users

```bash
# Change email
ipa user-mod jdoe --email=john.doe@example.com

# Change shell
ipa user-mod jdoe --shell=/bin/zsh

# Add SSH public key
ipa user-mod jdoe --sshpubkey="ssh-ed25519 AAAA... comment"

# Add a certificate (base64 DER, not base64 of the PEM text)
ipa user-add-cert jdoe --certificate="$(openssl x509 -outform der -in /path/to/user.crt | base64 -w 0)"

# Set account expiration date
ipa user-mod jdoe --principal-expiration=20270101000000Z

# Reset password (admin resets for user)
ipa passwd jdoe
```

### 1.4 User Lifecycle

```mermaid
graph TD
    A["Stage User<br/>cn=staged users"] -->|ipa stageuser-activate| B["Active User<br/>cn=users,cn=accounts"]
    B -->|ipa user-disable| C["Disabled User<br/>nsAccountLock=TRUE"]
    C -->|ipa user-enable| B
    B -->|ipa user-del --preserve| D["Preserved User<br/>cn=deleted users"]
    D -->|ipa user-undel| B
    B -->|ipa user-del| E["Permanently Deleted<br/>cannot restore"]


```

| State | LDAP location | Description |
|-------|--------------|-------------|
| **Staged** | `cn=staged users` | Pre-provisioned, cannot log in yet |
| **Active** | `cn=users,cn=accounts` | Normal active account |
| **Disabled** | `cn=users,cn=accounts` | `nsAccountLock=TRUE`, cannot log in |
| **Preserved** | `cn=deleted users` | Soft-deleted, retains UID/data, can restore |
| **Deleted** | (gone) | Permanently removed |

```bash
# Disable a user (locked, cannot log in)
ipa user-disable jdoe

# Re-enable
ipa user-enable jdoe

# Soft-delete (preserve) — retains uid/gid so the account can be restored.
# Preserve is opt-in. Plain `ipa user-del` removes the entry.
ipa user-del jdoe --preserve

# View preserved users
ipa user-find --preserved=true

# Restore a preserved user
ipa user-undel jdoe

# Permanently delete (no restore possible). This is the default.
ipa user-del jdoe
```

[↑ Back to TOC](#table-of-contents)

---

## 2. Group Management

### 2.1 Group Types

IPA supports three group types, plus a "protected" property on certain built-ins:

| Type | Description | Use case |
|------|-------------|---------|
| **POSIX** | Has `gidNumber`, visible to OS | Linux group membership, file permissions |
| **Non-POSIX** | No GID, LDAP-only | Email lists, policy grouping, AD trust |
| **External** | Contains AD SIDs as members | AD trust group mapping |
| **Protected/built-in** | A property of certain default groups (`admins`, `ipausers`), not a separate type — they are POSIX groups that cannot be deleted | System groups |

### 2.2 Creating and Managing Groups

```bash
# Create a POSIX group (default)
ipa group-add developers --desc="Development team"

# Create a non-POSIX group
ipa group-add maillist-dev --nonposix --desc="Dev mailing list"

# Add user to group
ipa group-add-member developers --users=jdoe

# Add multiple users at once
ipa group-add-member developers --users=jdoe,jsmith,amartinez

# Remove user from group
ipa group-remove-member developers --users=jdoe

# Show group details
ipa group-show developers

# Find groups a user belongs to
ipa user-show jdoe --all | grep memberof

# List all groups
ipa group-find
```

### 2.3 Nested Groups

IPA supports groups-within-groups (nested membership):

```bash
# Add a group as a member of another group
ipa group-add-member engineering --groups=developers
ipa group-add-member engineering --groups=ops
ipa group-add-member engineering --groups=qa
```

```mermaid
graph TD
    ENG[engineering group] --> DEV[developers group]
    ENG --> OPS[ops group]
    ENG --> QA[qa group]
    DEV --> U1[jdoe]
    DEV --> U2[jsmith]
    OPS --> U3[bwilson]
    QA --> U4[amartinez]
    QA --> U2
```

> 📝 SSSD resolves nested group membership automatically. A user in `developers`
> who is also in `engineering` via nesting will be shown as a member of both when
> you run `id jdoe` on a client.

[↑ Back to TOC](#table-of-contents)

---

## 3. Automember Rules

### 3.1 How Automember Works

Automember rules automatically add users (or hosts) to groups when they are
created, based on matching attributes. This eliminates the need to manually
assign group memberships for new users.

```mermaid
flowchart TD
    A["New user created<br/>ipa user-add"] --> B["IPA evaluates<br/>automember rules"]
    B --> C{"User attribute<br/>matches any rule?"}
    C -->|Rule: mail=.*@dev.example.com| D[Add to developers group]
    C -->|Rule: title=Manager| E[Add to managers group]
    C -->|Rule: dept=Engineering| F[Add to engineering group]
    C -->|No match| G["Add to default group<br/>if configured"]
    D --> H["User created with<br/>group memberships"]
    E --> H
    F --> H
    G --> H
```

### 3.2 Configuring Automember Rules

```bash
# Create an automember rule for a user group
ipa automember-add developers --type=group

# Add a condition: users whose uid matches regex
ipa automember-add-condition developers \
  --type=group \
  --key=uid \
  --inclusive-regex='^dev-'

# Add a condition: users with a specific email domain
ipa automember-add-condition developers \
  --type=group \
  --key=mail \
  --inclusive-regex='.*@dev\.example\.com$'

# Add an exclusive condition (exclude contractors)
ipa automember-add-condition employees \
  --type=group \
  --key=uid \
  --exclusive-regex='^contractor-'

# Set a default group (catch-all for no match)
ipa automember-default-group-set --type=group --default-group=ipausers

# Find orphaned automember rules (rules pointing at groups that no longer exist).
# Note: this is NOT a dry-run preview — there is no preview command. automember-rebuild
# applies rules retroactively to existing entries with no "what-if" mode.
ipa automember-find-orphans --type=group

# Rebuild automember for existing users (apply rules retroactively)
ipa automember-rebuild --type=group
ipa automember-rebuild --type=hostgroup
```

[↑ Back to TOC](#table-of-contents)

---

## 4. ID Ranges

ID ranges control which UIDs and GIDs are assigned to users and groups in FreeIPA.
This prevents UID/GID collisions between IPA domains, replicas, and AD trust domains.

### 4.1 DNA ID Ranges

The **Distributed Numeric Assignment (DNA)** plugin in 389-DS automatically assigns
UIDs/GIDs to new users. Each replica has its own non-overlapping range.

```mermaid
graph LR
    subgraph "IPA Domain UID Range: 1000000 - 1999999"
        R1["ipa1 DNA range<br/>1000000 - 1199999"]
        R2["ipa2 DNA range<br/>1200000 - 1399999"]
    end
    R1 -->|assigned to users| U1["jdoe uid=1000001<br/>jsmith uid=1000002"]
    R2 -->|assigned to users| U2["bwilson uid=1200001"]
```

> 📝 The domain ID range (`EXAMPLE.COM_id_range`, type `ipa-local`) is one pool. The default size is 200000 IDs at a random base. DNA only splits that pool across replicas. A second replica does not receive another 200000 IDs. The round numbers in the diagram are illustrative. Use `ipa idrange-find` and the DNA plugin config to see the live values.

```bash
# View current ID range
ipa idrange-find

# Show detailed range info
ipa idrange-show "EXAMPLE.COM_id_range"

# View DNA plugin config (389-DS level) — the ranges live under the posix-ids entry,
# not on the plugin root, so point the base there (sub scope finds the config entries)
ldapsearch -Y GSSAPI \
  -b "cn=posix-ids,cn=Distributed Numeric Assignment Plugin,cn=plugins,cn=config" \
  -s sub "(objectClass=*)"
```

### 4.2 Trust ID Ranges

When you establish an AD trust, a separate ID range is created for AD users. This
ensures AD users' UIDs/GIDs do not collide with local IPA users:

```mermaid
graph TD
    A[UID ranges] --> B["Local IPA users<br/>1000000 - 1999999<br/>type: local"]
    A --> C["AD trust users<br/>(AD.EXAMPLE.COM)<br/>2000000 - 2999999<br/>type: ipa-ad-trust"]
    B --> D["jdoe<br/>uid=1000001"]
    C --> E["AD\\jsmith<br/>uid=2000001"]
```

```bash
# View all ID ranges (local + trust)
ipa idrange-find --all

# Trust range is created by 'ipa trust-add --range-type=ipa-ad-trust'.
# ipa-ad-trust UIDs are derived from the AD SID inside that fence. They are
# not base+1, and they are not allocated by DNA.
# Do not change --base-id on a live trust range. That renumbers every mapped
# UID and breaks file ownership.
ipa idrange-find --all
```

[↑ Back to TOC](#table-of-contents)

---

## 5. ID Views

ID views allow you to **override POSIX attributes** (uid, gid, home dir, shell,
SSH keys) for specific users on specific hosts or host groups. This is useful for:
- Migration: keeping legacy UIDs on old servers while using IPA UIDs on new servers
- Per-host customisation: different shell or home dir on specific systems
- AD user overrides: assigning custom UIDs to AD trust users

```mermaid
graph LR
    A["IPA User: jdoe<br/>uid=1000001<br/>shell=/bin/bash"] -->|Default view| B["web01.example.com<br/>uid=1000001"]
    A -->|Legacy view| C["legacy01.example.com<br/>uid=500<br/>shell=/bin/sh"]
    A -->|Dev view| D["dev01.example.com<br/>homeDirectory=/workspace/jdoe"]
```

```bash
# Create an ID view
ipa idview-add legacy-view --desc="Legacy UID mappings for old servers"

# Add a user override in the view
ipa idoverrideuser-add legacy-view jdoe \
  --uid=500 \
  --gidnumber=500 \
  --shell=/bin/sh \
  --homedir=/home/jdoe

# Apply the view to a host
ipa idview-apply legacy-view --hosts=legacy01.example.com

# Apply to a host group
ipa idview-apply legacy-view --hostgroups=legacy-servers

# Show current view applied to a host
ipa idview-show legacy-view
```

[↑ Back to TOC](#table-of-contents)

---

## 6. Staged Users and User Lifecycle

**Staged users** are pre-provisioned accounts that exist in IPA but cannot
authenticate until activated. This is useful for HR/provisioning workflows where
accounts are created in advance.

```bash
# Create a staged user
ipa stageuser-add newemployee \
  --first=New --last=Employee \
  --email=newemployee@example.com

# List staged users
ipa stageuser-find

# Activate a staged user (moves to cn=users,cn=accounts)
ipa stageuser-activate newemployee

# Show staged user
ipa stageuser-show newemployee
```

```mermaid
sequenceDiagram
    participant HR as HR System
    participant IPA as FreeIPA
    participant User as New Employee

    HR->>IPA: ipa stageuser-add<br/>(2 weeks before start date)
    IPA-->>HR: Staged user created<br/>(cannot log in)
    Note over IPA: Account exists but inactive
    HR->>IPA: ipa stageuser-activate<br/>(on start date)
    IPA-->>HR: User moved to active<br/>Kerberos principal created
    IPA->>User: Welcome email / temp password
    User->>IPA: kinit newemployee<br/>(forced password change)
    IPA-->>User: Account fully active
```

[↑ Back to TOC](#table-of-contents)

---

## 7. Password Policies

IPA password policies define complexity, history, lockout, and expiry rules.
Policies are applied at the **global** level or to specific **groups**.

> 📖 **See also:** [Module 04, Section 8](04_authentication_kerberos.md#8-password-policies-and-lockout)
> for how password policies interact with Kerberos ticket lifetimes, account
> lockout counters, and per-user policy priority resolution.

```mermaid
graph TD
    A["Global Password Policy<br/>applies to all users"] -->|overridden by| B["Group Policy<br/>higher priority wins"]
    B --> C["User in 'admins' group<br/>stronger policy"]
    B --> D["User in 'contractors' group<br/>different expiry"]
    A --> E["User with no group policy<br/>uses global"]
```

```bash
# Show global password policy
ipa pwpolicy-show

# Modify global policy
# Options: minlife (min days before change), maxlife (days until expiry),
#          history (old passwords remembered), minclasses (upper/lower/digit/special),
#          minlength, maxfail (attempts before lockout),
#          failinterval (window in seconds), lockouttime (seconds, 0=permanent)
ipa pwpolicy-mod \
  --minlife=1 \
  --maxlife=90 \
  --history=10 \
  --minclasses=3 \
  --minlength=12 \
  --maxfail=5 \
  --failinterval=60 \
  --lockouttime=600

# Create a policy for a specific group
# priority: lower number = higher priority
ipa pwpolicy-add admins-policy \
  --group=admins \
  --priority=1 \
  --minlength=16 \
  --minclasses=4 \
  --maxlife=60 \
  --history=24

# Show all policies
ipa pwpolicy-find

# Check effective policy for a user
ipa pwpolicy-show --user=jdoe
```

[↑ Back to TOC](#table-of-contents)

---

## 8. Lab — User and Group Operations

```bash
# ── STEP 1: Create users ──────────────────────────────────────────────────────

kinit admin

ipa user-add alice \
  --first=Alice --last=Smith \
  --email=alice@example.com \
  --password

ipa user-add bob \
  --first=Bob --last=Jones \
  --email=bob@example.com \
  --password

ipa user-add carol \
  --first=Carol --last=White \
  --email=carol@example.com \
  --password

# ── STEP 2: Create groups ─────────────────────────────────────────────────────

ipa group-add developers --desc="Development Team"
ipa group-add ops --desc="Operations Team"
ipa group-add engineering --desc="Engineering Umbrella"

# ── STEP 3: Add members ───────────────────────────────────────────────────────

ipa group-add-member developers --users=alice --users=bob
ipa group-add-member ops --users=carol
ipa group-add-member engineering --groups=developers --groups=ops

# ── STEP 4: Verify membership ─────────────────────────────────────────────────

ipa user-show alice --all | grep -i memberof
ipa group-show engineering

# ── STEP 5: Automember rule ───────────────────────────────────────────────────

ipa group-add contractors --desc="Contractor accounts"
ipa automember-add contractors --type=group
ipa automember-add-condition contractors \
  --type=group \
  --key=uid \
  --inclusive-regex='^c-'

# Test: create a contractor
ipa user-add c-vendor1 --first=Vendor --last=One --password
ipa group-show contractors   # should contain c-vendor1 automatically

# ── STEP 6: Password policy ───────────────────────────────────────────────────

ipa pwpolicy-show
ipa pwpolicy-add dev-policy \
  --group=developers \
  --priority=10 \
  --minlength=14 \
  --maxlife=60 \
  --history=12

ipa pwpolicy-show --user=alice  # should show dev-policy

# ── STEP 7: User lifecycle ────────────────────────────────────────────────────

# Disable bob
ipa user-disable bob
ipa user-show bob | grep "Account disabled"

# Re-enable bob
ipa user-enable bob

# Soft-delete carol (preserved). Without --preserve the entry is gone
# and user-undel cannot restore it.
ipa user-del carol --preserve
ipa user-find --preserved=true

# Restore carol
ipa user-undel carol
ipa user-show carol
```


---

## Key Takeaways

- Identity data is the foundation for HBAC, sudo, RBAC, and certificate workflows.
- Consistent UID/GID planning avoids conflict later in migrations and trusts.
- Automember rules reduce manual administration when enrollment scales out.
- The next modules build directly on the users and groups created here.

[↑ Back to TOC](#table-of-contents)

---

*Licensed under [CC BY-NC-SA 4.0](LICENSE.md) · © 2026 UncleJS*