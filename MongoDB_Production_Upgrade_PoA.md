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
- Minor or patch upgrades (example: 4.2.x → 4.4.x, 6.0.x → 6.0.y)
- Linux operating systems (RHEL, CentOS, Amazon Linux, Ubuntu)

⚠️ MongoDB supports upgrading only one major version at a time.

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
5. Upgrade Feature Compatibility Version (FCV) after approval  

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
```
rs.status()
```

### Feature Compatibility Version
```
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
```

### Backup Validation
- Confirm recent full backup exists
- Validate restore procedure if required

---

## 6. Upgrade Execution Steps (Rolling Upgrade)

### Upgrade SECONDARY Node
```
systemctl stop mongod
systemctl start mongod
```

Validate:
```
mongod --version
rs.status()
```

Repeat for all SECONDARY nodes.

---

### Step Down PRIMARY
```
rs.stepDown(300)
```

---

### Upgrade Former PRIMARY
Follow the same steps as SECONDARY upgrade.

---

## 7. Post-Upgrade Validation
```
rs.status()
tail -n 200 /var/log/mongodb/mongod.log
```

---

## 8. Feature Compatibility Version (FCV) Upgrade
⚠️ Perform only after validation and approval.

```
db.adminCommand({ setFeatureCompatibilityVersion: "<TARGET_VERSION>" })
```

Verify:
```
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
```

---

## 9. Rollback Plan
Rollback is supported only before FCV upgrade.

Steps:
1. Stop MongoDB
2. Downgrade packages
3. Start MongoDB
4. Restore from backup if required

Rollback is NOT supported after FCV upgrade.

---

## 10. Reusability
This document can be reused for any MongoDB minor or patch upgrade by updating:
- MongoDB version
- Hostnames / IPs
- OS repository details

---

## 11. Change Log
| Date | Change | Author |
|------|--------|--------|
| YYYY-MM-DD | Initial reusable upgrade template | DBA Team |
