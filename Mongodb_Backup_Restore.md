# MongoDB Backup and Restore Guide

This document provides generic syntax and examples for taking backups and restoring MongoDB collections using `mongodump` and `mongorestore`. The commands below are written in a generic format so they can be reused in different environments.

---

## 1. Backup a MongoDB Collection with Query Filter

Use `mongodump` when you need to export specific documents from a collection using a query filter.

### **Generic Syntax**

```bash
mongodump \
  --host=<HOSTNAME_OR_IP> \
  --port=<PORT> \
  -u <USERNAME> \
  -p '<PASSWORD>' \
  --authenticationDatabase=<AUTH_DB> \
  --db=<DATABASE_NAME> \
  --collection=<COLLECTION_NAME> \
  --query='<QUERY_IN_JSON>' \
  --out=<OUTPUT_DIRECTORY_PATH>
```

### Explanation

* **--host / --port**: MongoDB server address.
* **-u / -p**: Credentials of a user with read access.
* **--authenticationDatabase**: Database where the user is created.
* **--db**: Target database.
* **--collection**: Specific collection to export.
* **--query**: Optional filter to export only selected documents.
* **--out**: Directory where the dump will be stored.

---

## 2. Restore a Specific BSON File into a Target Collection

When the dump structure does not follow the standard MongoDB folder layout, you must explicitly mention the database and collection during restore.

### **Generic Syntax**

```bash
mongorestore \
  --uri="mongodb://<USERNAME>:<PASSWORD>@<HOST>:<PORT>/?authSource=<AUTH_DB>" \
  --nsInclude="<DATABASE_NAME>.<COLLECTION_NAME>" \
  --collection=<COLLECTION_NAME> \
  --db=<DATABASE_NAME> \
  <PATH_TO_BSON_FILE>
```

### Explanation

* **--uri**: Connection string including authentication details.
* **--nsInclude**: Defines the namespace (database.collection) for restore.
* **--collection**: Collection to restore data into.
* **--db**: Target database name.
* **Final argument**: Path of the `.bson` file to restore.

---

## Notes

* Ensure that the user has the required roles for backup (read) and restore (write).
* Use appropriate quoting for passwords containing special characters.
* For full database dumps, use only `--db` without `--collection` or `--query`.
* For full cluster backups or replica sets, `--uri` with the full connection string is recommended.

---

This guide can be uploaded directly into GitHub as a Markdown file to standardize MongoDB backup and restore procedures across environments.
