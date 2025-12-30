# MongoDB Production Upgrade – Plan of Action (PoA)

## Overview
This document provides a standardized and reusable Plan of Action (PoA) for performing MongoDB production replica set upgrades using a rolling upgrade strategy.

It is intended for minor and patch version upgrades and follows MongoDB best practices to ensure:

- Zero data loss
- No unplanned downtime
- Minimal application impact
- Safe rollback capability (until FCV upgrade)

---

## 1. Scope & Applicability
This PoA applies to:

- MongoDB Replica Set deployments
- Minor or patch upgrades  
  Examples: `4.2.x → 4.4.x`, `6.0.x → 6.0.y`
- Linux operating systems  
  RHEL, CentOS, Amazon Linux, Ubuntu

**Note:** MongoDB supports upgrading only one major version at a time.

---

## 2. Upgrade Objectives
- Safely upgrade MongoDB binaries to the target version
- Maintain replica set availability throughout the activity
- Avoid application downtime (except brief primary step-down)
- Retain rollback capability until FCV upgrade

---

## 3. Upgrade Strategy (Best Practice)
A rolling upgrade approach will be followed:

1. Upgrade SECONDARY nodes one at a time
2. Perform a controlled PRIMARY step-down
3. Upgrade the former PRIMARY
4. Validate cluster health
5. Upgrade Feature Compatibility Version (FCV) --after approval

---

## 4. Roles & Responsibilities

### Client Responsibilities – Before Upgrade
- Approve maintenance window
- Confirm application can tolerate brief primary election
- Ensure application-level backups (if required)
- Provide monitoring and alerting access

### Client Responsibilities – After Upgrade
- Perform application smoke testing
- Confirm application functionality
- Approve FCV upgrade

---

## 5. Pre-Upgrade Checklist (Mandatory)

### Replica Set Health
```js
rs.status()
```

### Feature Compatibility Version
```js
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
```

### Backup Validation
- Confirm recent full backup exists
- Validate restore procedure if required

---

## 6. Upgrade Execution Steps (Rolling Upgrade)

### Step 1: Upgrade SECONDARY Node

#### Check MongoDB Version
```bash
mongod --version
```

#### Check MongoDB Service Status
```bash
systemctl status mongod
```

#### Stop MongoDB
```bash
systemctl stop mongod
systemctl status mongod
```

### Configure MongoDB Repository (Example: RHEL 8)
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/mongodb-org.repo
[mongodb-org]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/8/mongodb-org/<TARGET_VERSION>/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-<TARGET_VERSION>.asc
EOF
```

### Install Target Version
```bash
yum install -y \
mongodb-org-<VERSION> \
mongodb-org-server-<VERSION> \
mongodb-org-shell-<VERSION> \
mongodb-org-mongos-<VERSION> \
mongodb-org-tools-<VERSION>
```

### Start MongoDB
```bash
systemctl start mongod
```

### Validation
```bash
mongod --version
```

```js
rs.status()
```

Ensure the node rejoins the replica set as **SECONDARY** before proceeding.

---

### Step 2: Repeat for Remaining SECONDARY Nodes
- Upgrade only one SECONDARY at a time
- Validate replica set health after each upgrade

---

### Step 3: Step Down PRIMARY
```js
rs.stepDown(300)
```

---

### Step 4: Upgrade Former PRIMARY
- Stop MongoDB
- Upgrade MongoDB packages
- Start MongoDB
- Validate replica set health

(Same steps as SECONDARY upgrade)

---

## 7. Post-Upgrade Validation

```js
rs.status()
```

```js
db.version()
```

```js
rs.status().members.map(m => ({ name: m.name, state: m.stateStr }))
```

```js
db.hello()
```
Verify `"isWritablePrimary": true` on the PRIMARY.

```js
rs.printSecondaryReplicationInfo()
```

```bash
tail -n 200 /var/log/mongodb/mongod.log
```

---

## 8. Feature Compatibility Version (FCV) Upgrade
Perform only after validation and client approval.

```js
db.adminCommand({ setFeatureCompatibilityVersion: "<TARGET_VERSION>" })
```

### Verify FCV
```js
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
```

---

## 9. Rollback Plan

Rollback is supported **only before FCV upgrade**.

### Rollback Steps
1. Stop MongoDB
2. Downgrade MongoDB packages
3. Start MongoDB
4. Restore from backup if required

**Note:** Rollback is **NOT supported** after FCV upgrade.
