```
#!/bin/bash
# MongoDB Archive + Cleanup Script (Production Safe)
# Manual Credentials Version (No AWS Secrets Manager)
# ==========================================================

# =========================
# MongoDB Connection Details
# =========================

USERNAME=""
PASSWORD=""
HOST=""
PORT=""
DATABASE=""
AUTH_DB=""

# =========================
# Cleanup Settings
# =========================

RETENTION_DAYS=
BATCH_SIZE=
SLEEP_SECONDS=1

# =========================
# Backup Location
# =========================

BACKUP_MOUNT=""
BASE_BACKUP_PATH="${BACKUP_MOUNT}/mongo_archive"
TODAY=$(date +%Y%m%d_%H%M%S)
ARCHIVE_DIR="${BASE_BACKUP_PATH}/${TODAY}"

LOG_FILE="/var/log/mongo_archive_cleanup.log"

echo "====================================================" | tee -a $LOG_FILE
echo "Archive + Cleanup started at $(date)" | tee -a $LOG_FILE
echo "Backup mount path: $BACKUP_MOUNT" | tee -a $LOG_FILE
echo "Archive directory: $ARCHIVE_DIR" | tee -a $LOG_FILE

# =========================
# Validate Backup Mount
# =========================

if ! mountpoint -q "$BACKUP_MOUNT"; then
    echo "ERROR: $BACKUP_MOUNT is not mounted. Exiting." | tee -a $LOG_FILE
    exit 1
fi

mkdir -p "$ARCHIVE_DIR"

# =========================
# Mongo Connection URI
# =========================

MONGO_URI="mongodb://${USERNAME}:${PASSWORD}@${HOST}:${PORT}/${DATABASE}?authSource=${AUTH_DB}"

# =========================
# Calculate Cutoff ObjectId
# =========================

CUTOFF_OID=$(printf "%08x" $(date -d "$RETENTION_DAYS days ago" +%s))"0000000000000000"
echo "Cutoff ObjectId: $CUTOFF_OID" | tee -a $LOG_FILE

collections=(
  ""
  ""
)

# =========================
# Process Collections
# =========================

for coll in "${collections[@]}"; do

  echo "----------------------------------------------------" | tee -a $LOG_FILE
  echo "Processing collection: $coll" | tee -a $LOG_FILE

  AVAILABLE_BYTES=$(df --output=avail -B1 "$BACKUP_MOUNT" | tail -1)
  echo "Available disk space (bytes): $AVAILABLE_BYTES" | tee -a $LOG_FILE

  # Count Documents
  DOC_COUNT=$(mongosh "$MONGO_URI" --quiet --eval "
    db.getCollection('$coll')
      .countDocuments({ _id: { \$lt: ObjectId('$CUTOFF_OID') } })
  ")

  if [ "$DOC_COUNT" -eq 0 ]; then
      echo "No documents eligible for archive in $coll" | tee -a $LOG_FILE
      continue
  fi

  echo "Documents eligible for archive: $DOC_COUNT" | tee -a $LOG_FILE

  # Estimate archive size
  ESTIMATED_SIZE=$(mongosh "$MONGO_URI" --quiet --eval "
    db.getCollection('$coll').aggregate([
      { \$match: { _id: { \$lt: ObjectId('$CUTOFF_OID') } } },
      { \$group: { _id: null, totalSize: { \$sum: { \$bsonSize: '\$\$ROOT' } } } }
    ]).toArray()[0]?.totalSize || 0
")

  REQUIRED_SPACE=$(echo "$ESTIMATED_SIZE * 1.2" | bc | awk '{printf "%.0f",$0}')
  echo "Estimated size (bytes): $ESTIMATED_SIZE" | tee -a $LOG_FILE
  echo "Required space with buffer: $REQUIRED_SPACE" | tee -a $LOG_FILE

  if [ "$REQUIRED_SPACE" -gt "$AVAILABLE_BYTES" ]; then
      echo "ERROR: Not enough disk space for $coll archive. Skipping." | tee -a $LOG_FILE
      continue
  fi

  # =========================
  # Archive Data
  # =========================

  mongodump \
    --uri="$MONGO_URI" \
    --collection="$coll" \
    --query="{ \"_id\": { \"\$lt\": { \"\$oid\": \"$CUTOFF_OID\" } } }" \
    --out="$ARCHIVE_DIR"

  if [ $? -ne 0 ]; then
      echo "ERROR: mongodump failed for $coll. Skipping delete." | tee -a $LOG_FILE
      continue
  fi

  # =========================
  # Verify Archive
  # =========================

  BSON_FILE="$ARCHIVE_DIR/$DATABASE/$coll.bson"

  if [ ! -f "$BSON_FILE" ] || [ ! -s "$BSON_FILE" ]; then
      echo "ERROR: Archive file missing or empty for $coll. Skipping delete." | tee -a $LOG_FILE
      continue
  fi

  echo "Archive verification successful for $coll" | tee -a $LOG_FILE

  # =========================
  # Delete Data in Batches Until All Deleted
  # =========================

  mongosh "$MONGO_URI" --quiet --eval "

    var cutoff = ObjectId('$CUTOFF_OID');
    var batchSize = $BATCH_SIZE;
    var totalDeleted = 0;

    while (true) {

        // Fetch a batch of _id values
        var ids = db.getCollection('$coll')
                    .find({ _id: { \$lt: cutoff } }, { _id: 1 })
                    .limit(batchSize)
                    .toArray()
                    .map(d => d._id);

        if (ids.length === 0) break; // No more documents to delete

        // Delete the batch
        var result = db.getCollection('$coll')
                       .deleteMany({ _id: { \$in: ids } });

        totalDeleted += result.deletedCount;

        print('Deleted batch of ' + result.deletedCount + ' from $coll. Total deleted so far: ' + totalDeleted);

        sleep($SLEEP_SECONDS * 1000); // sleep expects milliseconds in mongosh
    }

    print('Completed deletion for $coll. Total deleted: ' + totalDeleted);

  " | tee -a $LOG_FILE

done

echo "Archive + Cleanup completed at $(date)" | tee -a $LOG_FILE
echo "====================================================" | tee -a $LOG_FILE
```
