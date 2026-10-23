# CS-06 — Replication Cheatsheet
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](../LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![FreeIPA](https://img.shields.io/badge/FreeIPA-v4.12-blue)](https://www.freeipa.org)

> Quick-reference for FreeIPA replication management, topology, health checks, and recovery on RHEL 10.

---

> 🔁 **See also:** [Module 12 — Replication Topology](../12_replication_topology.md)

---

## Table of Contents

- [Topology Management](#topology-management)
- [Replica Install and Remove](#replica-install-and-remove)
- [Replication Health](#replication-health)
- [Force Sync and Re-initialize](#force-sync-and-re-initialize)
- [CA Replication](#ca-replication)
- [Backup and Restore](#backup-and-restore)
- [Conflict Resolution](#conflict-resolution)
- [Troubleshooting](#troubleshooting)

---

## Topology Management

```bash
# List all IPA servers
ipa server-find

# Show server details (services, roles)
ipa server-show ipa1.example.com

# List topology segments (domain suffix)
ipa topologysegment-find --topologysuffix=domain

# List topology segments (CA suffix)
ipa topologysegment-find --topologysuffix=ca

# Show a specific segment
# Segment names are auto-generated (host-to-host) — get the exact name from topologysegment-find
ipa topologysegment-show domain ipa1.example.com-to-ipa2.example.com

# Add a new topology segment
ipa topologysegment-add domain \
    --leftnode=ipa1.example.com \
    --rightnode=ipa3.example.com \
    --direction=both

ipa topologysegment-add ca \
    --leftnode=ipa1.example.com \
    --rightnode=ipa3.example.com \
    --direction=both

# Remove a topology segment (use the auto-generated name from topologysegment-find)
ipa topologysegment-del domain ipa2.example.com-to-ipa3.example.com

# Verify topology is connected (no isolated replicas)
ipa topologysuffix-verify domain
ipa topologysuffix-verify ca
# Expected: "Replication topology graph is connected."
```

[↑ Back to TOC](#table-of-contents)

---

## Replica Install and Remove

```bash
# Install new replica (run on the new host)
sudo ipa-replica-install \
    --setup-ca \
    --setup-dns \
    --forwarder=8.8.8.8 \
    --principal=admin \
    --admin-password='AdminPassword123!'

# Install without DNS
sudo ipa-replica-install \
    --setup-ca \
    --principal=admin \
    --admin-password='AdminPassword123!'

# Install without CA
sudo ipa-replica-install \
    --setup-dns \
    --forwarder=8.8.8.8 \
    --principal=admin \
    --admin-password='AdminPassword123!'

# Add DNS to existing replica
sudo ipa-dns-install \
    --forwarder=8.8.8.8

# Add CA to existing replica
sudo ipa-ca-install \
    --principal=admin \
    --admin-password='AdminPassword123!'

# Remove a replica cleanly (run on the replica being decommissioned)
# ⚠️ IRREVERSIBLE — take a backup first (ipa-backup)
sudo ipa-server-install --uninstall

# Remove from a remote server (if target is unreachable)
ipa server-del ipa-old.example.com

# Force remove (ignores connectivity issues)
# ⚠️ Can orphan RUVs/topology segments — follow up with clean-ruv to clear stale RUVs
ipa server-del ipa-old.example.com \
    --force \
    --ignore-topology-disconnect
```

[↑ Back to TOC](#table-of-contents)

---

## Replication Health

```bash
# Comprehensive health check
sudo ipa-healthcheck

# Replication-specific checks only
sudo ipa-healthcheck --source ipahealthcheck.ds.replication

# Check replication status
# NOTE: ipa-replica-manage and ipa-csreplica-manage are low-level legacy tools.
# They remain functional but the preferred approach is the ipa topology* API
# and direct 389-DS LDAP monitor queries (shown below).
sudo ipa-replica-manage -p 'DM_Password' list
sudo ipa-replica-manage -p 'DM_Password' status

# Check via 389-DS LDAP monitor
sudo ldapsearch -x -H ldap://localhost \
    -D "cn=Directory Manager" -W \
    -b "cn=monitor" \
    "(objectClass=nsds5replicationagreement)" \
    nsds5replicaLastUpdateStatus \
    nsds5replicaLastUpdateStart \
    nsds5replicaLastUpdateEnd \
    nsds5replicaUpdateInProgress 2>/dev/null | \
    grep -E "dn:|nsds5replica"

# Expected healthy status:
# nsds5replicaLastUpdateStatus: 0 Replica acquired successfully: Incremental update succeeded

# Check replication errors in 389-DS error log
sudo grep -iE "repl error|replication.*fail|consumer" \
    /var/log/dirsrv/slapd-EXAMPLE-COM/errors | tail -20

# Lag guidance:
# < 60s   = healthy target on a stable LAN
# 1-5 min = possible during bulk imports or temporary peer load
# > 5 min = investigate network, backlog, busy peers, or topology bottlenecks

# Check all agreements and their status
sudo ldapsearch -x -H ldap://localhost \
    -D "cn=Directory Manager" -W \
    -b "cn=config" \
    "(objectClass=nsDS5ReplicationAgreement)" \
    cn nsDS5ReplicaHost nsDS5ReplicaLastUpdateStatus | \
    grep -E "^(dn|cn|nsDS5)"
```

[↑ Back to TOC](#table-of-contents)

---

## Force Sync and Re-initialize

```bash
# Force immediate sync FROM a specific master
sudo ipa-replica-manage -p 'DM_Password' force-sync \
    --from=ipa1.example.com

# Re-initialize this replica from scratch
# (WARNING: destructive — rebuilds all data from source)
sudo ipa-replica-manage -p 'DM_Password' re-initialize \
    --from=ipa1.example.com

# Re-initialize CA suffix from scratch
sudo ipa-csreplica-manage -p 'DM_Password' re-initialize \
    --from=ipa1.example.com

# Monitor re-initialization progress
sudo ldapsearch -x -H ldap://localhost \
    -D "cn=Directory Manager" -W \
    -b "cn=monitor" \
    "(nsds5replicaUpdateInProgress=TRUE)" \
    nsds5replicaUpdateInProgress \
    nsds5replicaLastUpdateStatus 2>/dev/null

# Clean stale RUVs (after removing a replica)
# Find replica IDs
sudo ipa-replica-manage -p 'DM_Password' list

# Clean specific RUV
# ⚠️ Irreversible — confirm the replica-id maps to a genuinely removed replica before cleaning
sudo ipa-replica-manage -p 'DM_Password' clean-ruv <replica-id>

# Verify RUV cleanup
sudo ipa-replica-manage -p 'DM_Password' list
```

[↑ Back to TOC](#table-of-contents)

---

## CA Replication

```bash
# CA agreements are the "ca" topology suffix, not ipa-csreplica-manage.
ipa topologysegment-find ca
ipa topologysuffix-verify ca

# Rebuild one CA segment from the other side. This overwrites the target.
ipa topologysegment-reinitialize ca ipa1.example.com-to-ipa2.example.com --right

# CA renewal master and CRL generation master are different roles.
ipa config-show | grep "IPA CA renewal master"
ipa config-mod --ca-renewal-master-server=ipa2.example.com

# Move CRL generation with ipa-crlgen-manage. Disable the old master first.
# On the old CRL master:
ipa-crlgen-manage disable
# On the new CRL master:
ipa-crlgen-manage enable
ipa-crlgen-manage status

curl -s http://ipa2.example.com/ipa/crl/MasterCRL.bin | \
    openssl crl -inform DER -noout -lastupdate -nextupdate
```

[↑ Back to TOC](#table-of-contents)

---

## Backup and Restore

```bash
# Data-only online backup. Services stay up. This is not a full restore image.
sudo ipa-backup --data --online

# Full offline backup. ipa-backup stops and restarts IPA services itself.
sudo ipa-backup

# Backup to specific directory
sudo ipa-backup --dir=/mnt/nas/ipa-backups/

# List backups
ls -lh /var/lib/ipa/backup/

# Restore from backup
sudo ipa-restore /var/lib/ipa/backup/ipa-full-YYYY-MM-DD-HH-MM-SS

# Restore with DM password
sudo ipa-restore \
    --password='DMPassword123!' \
    /var/lib/ipa/backup/ipa-full-YYYY-MM-DD-HH-MM-SS

# After restoring a single master:
# Re-initialize all other replicas from the restored master
sudo ipa-replica-manage -p 'DM_Password' re-initialize \
    --from=ipa1.example.com

# Backup retention — keep last 7 backups, delete older
# Step 1: DRY RUN — preview what would be deleted
ls -dt /var/lib/ipa/backup/ipa-full-* | tail -n +8
# Step 2: verify the path and count look correct, then delete
# ⚠️ Double-check the path before running rm -rf
ls -dt /var/lib/ipa/backup/ipa-full-* | \
    tail -n +8 | \
    xargs -r sudo rm -rf
```

[↑ Back to TOC](#table-of-contents)

---

## Conflict Resolution

```bash
# Find replication conflict entries
sudo ldapsearch -x -H ldap://localhost \
    -D "cn=Directory Manager" -W \
    -b "dc=example,dc=com" \
    "(nsds5ReplConflict=*)" \
    dn nsds5ReplConflict uid 2>/dev/null

# Conflict entry DNs look like:
# nsuniqueid=<uuid>+uid=jsmith,cn=users,...

# Inspect both conflicting entries
sudo ldapsearch -x -H ldap://localhost \
    -D "cn=Directory Manager" -W \
    -b "dc=example,dc=com" \
    "(uid=jsmith)" uid cn mail

# Delete the conflict (wrong) entry
# ⚠️ Destructive — confirm which of the two entries is the conflict before deleting
sudo ldapdelete -x -H ldap://localhost \
    -D "cn=Directory Manager" -W \
    "nsuniqueid=<uuid>+uid=jsmith,cn=users,cn=accounts,dc=example,dc=com"

# Find tombstone entries (deleted objects)
sudo ldapsearch -x -H ldap://localhost \
    -D "cn=Directory Manager" -W \
    -b "dc=example,dc=com" \
    "(objectClass=nsTombstone)" \
    dn nsds5ReplConflict 2>/dev/null | head -20
```

[↑ Back to TOC](#table-of-contents)

---

## Troubleshooting

```bash
# Replication stopped / stuck
sudo ldapsearch -x -H ldap://localhost \
    -D "cn=Directory Manager" -W \
    -b "cn=monitor" \
    "(objectClass=nsds5replicationagreement)" \
    nsds5replicaLastUpdateStatus 2>/dev/null | \
    grep -v "^#\|^$"

# Status code meanings:
# 0  = OK
# 1  = Busy (will retry)
# 19 = Busy
# 81 = Can't contact LDAP (network issue)
# 82 = TLS/cert error
# -1 = Fatal error

# Fix: restart 389-DS on both replicas
sudo systemctl restart dirsrv@EXAMPLE-COM.service

# Topology disconnected
ipa topologysuffix-verify domain
# If disconnected — add missing segments:
ipa topologysegment-add domain \
    --leftnode=isolated.example.com \
    --rightnode=ipa1.example.com \
    --direction=both

# Healthcheck for replication
sudo ipa-healthcheck --source ipahealthcheck.ds.replication

# Check for clock skew between replicas
for server in ipa1.example.com ipa2.example.com; do
    echo -n "$server: "
    ssh "$server" "date '+%Y-%m-%d %H:%M:%S %Z'"
done
```

[↑ Back to TOC](#table-of-contents)

---

*Licensed under [CC BY-NC-SA 4.0](../LICENSE.md) · © 2026 UncleJS*
