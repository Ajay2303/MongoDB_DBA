# MongoDB Sequential Upgrade Guide (3.4 to 8.x)

## Environment Details
- Operating System: CentOS Linux 7
- MongoDB Edition: Community
- Upgrade Type: In-place Sequential Major Version Upgrade
- Package Manager: YUM

---

## Overview

MongoDB supports only sequential major version upgrades. Skipping versions can result in startup failures, data corruption, or index incompatibilities. This document provides a production-grade, step-by-step procedure to upgrade MongoDB from version 3.4 up to 8.x using supported upgrade paths.

Upgrade Path:

3.4 → 3.6 → 4.0 → 4.2 → 4.4 → 5.0 → 6.0 → 7.0 → 8.0

---

## Pre-Upgrade Checklist (Mandatory)

Before starting any upgrade:

- Take a full database backup (mongodump or filesystem snapshot)
- Verify replica set health (if applicable)
- Ensure sufficient disk space (minimum 2x data size)
- Validate application compatibility
- Do not skip Feature Compatibility Version (FCV)
- Do not skip MongoDB versions
- Do not modify MongoDB binary symlink paths

---

## Version Verification Commands

Check MongoDB binary version:
```
mongod --version
```

Check Feature Compatibility Version:
```
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
```

---

## MongoDB 3.4 to 3.6 Upgrade

### Create Repository
```
cat <<EOF > /etc/yum.repos.d/mongodb-org-3.6.repo
[mongodb-org-3.6]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/7/mongodb-org/3.6/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.6.asc
EOF
```

### Upgrade Steps
```
systemctl stop mongod
yum install -y mongodb-org
systemctl daemon-reload
systemctl start mongod
```

### Set FCV
```
mongo --eval 'db.adminCommand({ setFeatureCompatibilityVersion: "3.6" })'
```

---

## MongoDB 3.6 to 4.0 Upgrade

### Create Repository
```
cat <<EOF > /etc/yum.repos.d/mongodb-org-4.0.repo
[mongodb-org-4.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/7/mongodb-org/4.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc
EOF
```

### Upgrade Steps
```
systemctl stop mongod
yum install -y mongodb-org
systemctl daemon-reload
systemctl start mongod
```

### Set FCV
```
mongo --eval 'db.adminCommand({ setFeatureCompatibilityVersion: "4.0" })'
```

---

## MongoDB 4.0 to 4.2 Upgrade

### Create Repository
```
cat <<EOF > /etc/yum.repos.d/mongodb-org-4.2.repo
[mongodb-org-4.2]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/7/mongodb-org/4.2/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.2.asc
EOF
```

### Upgrade Steps
```
systemctl stop mongod
yum install -y mongodb-org
systemctl daemon-reload
systemctl start mongod
```

### Set FCV
```
mongo --eval 'db.adminCommand({ setFeatureCompatibilityVersion: "4.2" })'
```

---

## MongoDB 4.2 to 4.4 Upgrade

### Create Repository
```
cat <<EOF > /etc/yum.repos.d/mongodb-org-4.4.repo
[mongodb-org-4.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/7/mongodb-org/4.4/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.4.asc
EOF
```

### Upgrade Steps
```
systemctl stop mongod
yum install -y mongodb-org
systemctl daemon-reload
systemctl start mongod
```

### Set FCV
```
mongo --eval 'db.adminCommand({ setFeatureCompatibilityVersion: "4.4" })'
```

---

## MongoDB 5.0 and Above (Important Note)

MongoDB versions **5.0 and above do not support CentOS Linux 7**.  
Before proceeding beyond MongoDB 4.4, the operating system **must be upgraded** to one of the following supported platforms:

- Rocky Linux 8 / 9
- AlmaLinux 8 / 9
- Amazon Linux 2023
- RHEL 8 / 9

Proceeding without an OS upgrade may result in installation failure or unstable behavior.

---

## MongoDB 4.4 to 5.0 Upgrade

### Create Repository

```bash
cat <<EOF > /etc/yum.repos.d/mongodb-org-5.0.repo
[mongodb-org-5.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/8/mongodb-org/5.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-5.0.asc
EOF
```
### Upgrade MongoDB Binaries
```
systemctl stop mongod
yum install -y mongodb-org
systemctl daemon-reload
systemctl start mongod
```
### Set FCV
```
db.adminCommand({ setFeatureCompatibilityVersion: "5.0" })
```

Key Enhancements:
- Time-series collections
- Resharding support
- Improved transaction handling

---

## MongoDB 5.0 to 6.0
### Create Repository
```
cat <<EOF > /etc/yum.repos.d/mongodb-org-6.0.repo
[mongodb-org-6.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/8/mongodb-org/6.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-6.0.asc
EOF
```
### Upgrade MongoDB Binaries
```
systemctl stop mongod
yum install -y mongodb-org
systemctl daemon-reload
systemctl start mongod
```
### Set FCV
```
db.adminCommand({ setFeatureCompatibilityVersion: "6.0" })
```

Key Enhancements:
- Queryable encryption
- Cluster-to-cluster sync
- Improved aggregation framework

---

## MongoDB 6.0 to 7.0
### Create Repository
```
cat <<EOF > /etc/yum.repos.d/mongodb-org-7.0.repo
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/8/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-7.0.asc
EOF
```
### Upgrade MongoDB Binaries
```
systemctl stop mongod
yum install -y mongodb-org
systemctl daemon-reload
systemctl start mongod
```
### Set FCV
```
db.adminCommand({ setFeatureCompatibilityVersion: "7.0" })
```

Key Enhancements:
- Columnstore indexes
- Replication performance improvements
- Enhanced query optimizer

---

## MongoDB 7.0 to 8.0
### Create Repository
```
cat <<EOF > /etc/yum.repos.d/mongodb-org-8.0.repo
[mongodb-org-8.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/8.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-8.0.asc
EOF
```
### Upgrade MongoDB Binaries
```
systemctl stop mongod
yum install -y mongodb-org
systemctl daemon-reload
systemctl start mongod
```
### Set FCV
```
db.adminCommand({ setFeatureCompatibilityVersion: "8.0" })
```

Key Enhancements:
- Advanced indexing
- Improved sharding performance
- Enhanced workload isolation

---

## Post-Upgrade Validation

After each upgrade:

```
mongod --version
```

```
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
```

---

## Log Validation

Check MongoDB logs after each startup:
```
tail -n 50 /var/log/mongodb/mongod.log
```

Verify there are no errors related to:
- WiredTiger
- FCV mismatches
- Index rebuild failures
- Authentication issues

---

## Critical Rules

- Feature Compatibility Version must always match the running MongoDB binary
- Never downgrade FCV
- Never skip major versions
- Always test upgrades in lower environments first

---

## Author
Ajay S  
Database Engineer  
MongoDB Community Edition
