# MongoDB Collection-Wise Backup Script with S3 Upload & Email Report

This script performs **collection-wise MongoDB backups**, compresses them, uploads them to **Amazon S3**, and sends a **summary email report** with logs.  
It is designed for **daily automated backups**.

---

##  Features
- Collection-wise backup using `mongodump`
- Excludes system views
- Uploads backups to Amazon S3
- Sends email reports (HTML format: Green for success, Red for failure)
- Logs all activity to `/var/log/mongodb/`
- Cleans up local backups after successful upload
- Supports **password file** for better security

---

##  Prerequisites
- `mongodump` and `mongosh` installed  
- AWS CLI configured with proper IAM permissions  
- `mailx` installed for email notifications  
- MongoDB user with read access to target database  

---

## Script

```bash
# === USER CONFIGURATION ===
MONGO_PORT=""
MONGO_USER=""
MONGO_PASS=""
AUTH_DB=""
DB_NAME=""

S3_BUCKET=""
LOCAL_BACKUP_DIR="/mnt/database/backup"
EMAIL_TO=""

# === SYSTEM PATH UPDATE ===
export PATH=$PATH:/usr/local/bin:/usr/bin

# === INTERNAL CONFIGURATION ===
START_TIME=$(date +%s)
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
TODAY_DIR="${LOCAL_BACKUP_DIR}/${TIMESTAMP}"
S3_FOLDER="${DB_NAME}_${TIMESTAMP}"
LOG_FILE="/var/log/mongodb/collection_backup_log_$TIMESTAMP.log"

mkdir -p "$TODAY_DIR"
mkdir -p "$(dirname "$LOG_FILE")"
touch "$LOG_FILE"

echo "MongoDB Backup started at: $(date)" | tee -a "$LOG_FILE"
ALL_SUCCESS=true
FAILED_COLLECTIONS=()

# === FETCH COLLECTIONS (Exclude Views) ===
COLLECTIONS=$(mongosh \
  --port "$MONGO_PORT" \
  --username "$MONGO_USER" \
  --password "$MONGO_PASS" \
  --authenticationDatabase "$AUTH_DB" \
  --quiet --eval "db.getCollectionInfos({ type: 'collection' }).map(c => c.name).filter(n => n !== 'system.views').join(' ')" "$DB_NAME" 2>>"$LOG_FILE")

TOTAL_COLLECTIONS=$(echo "$COLLECTIONS" | wc -w)

if [[ -z "$COLLECTIONS" ]]; then
    echo "Failed to fetch collections from $DB_NAME" | tee -a "$LOG_FILE"
    exit 1
fi

# === BACKUP EACH COLLECTION TO LOCAL ===
for COLLECTION in $COLLECTIONS; do
    BACKUP_NAME="${DB_NAME}.${COLLECTION}.${TIMESTAMP}.gz"
    LOCAL_FILE="${TODAY_DIR}/${BACKUP_NAME}"

    echo "Backing up $DB_NAME.$COLLECTION" | tee -a "$LOG_FILE"

    mongodump \
        --port "$MONGO_PORT" \
        --username "$MONGO_USER" \
        --password "$MONGO_PASS" \
        --authenticationDatabase "$AUTH_DB" \
        --db "$DB_NAME" \
        --collection "$COLLECTION" \
        --gzip \
        --archive="$LOCAL_FILE"

    if [ $? -eq 0 ]; then
        echo "SUCCESS: $COLLECTION" | tee -a "$LOG_FILE"
    else
        echo "FAILED: $COLLECTION" | tee -a "$LOG_FILE"
        ALL_SUCCESS=false
        FAILED_COLLECTIONS+=("$COLLECTION")
    fi
done

# === STEP 2: UPLOAD TO S3 ===
if [ "$ALL_SUCCESS" = true ]; then
    aws s3 cp "$TODAY_DIR" "$S3_BUCKET/$S3_FOLDER/" --recursive >> "$LOG_FILE" 2>&1
    if [ $? -eq 0 ]; then
        echo "Upload SUCCESS to S3" | tee -a "$LOG_FILE"
        rm -rf "$TODAY_DIR"
        CLEANUP_MSG="Local backup directory was removed after successful S3 upload."
    else
        echo "Upload FAILED to S3" | tee -a "$LOG_FILE"
        ALL_SUCCESS=false
    fi
fi

# === STEP 3: BACKUP METRICS ===
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))
DURATION_HUMAN="$(printf '%02dh:%02dm:%02ds' $((DURATION/3600)) $((DURATION%3600/60)) $((DURATION%60)))"

S3_STATS=$(aws s3 ls $S3_BUCKET/$S3_FOLDER/ --recursive --summarize)
S3_OBJECTS=$(echo "$S3_STATS" | grep "Total Objects" | awk '{print $3}')
S3_SIZE_BYTES=$(echo "$S3_STATS" | grep "Total Size" | awk '{print $3}')
S3_SIZE_HUMAN=$(numfmt --to=iec --suffix=B $S3_SIZE_BYTES)

# === STEP 4: EMAIL REPORT (HTML with Colors) ===
DB_INFO="Database: <b>$DB_NAME</b><br>
S3 Path: <b>$S3_BUCKET/$S3_FOLDER</b><br>
Timestamp: <b>$TIMESTAMP</b>"

if [ "$ALL_SUCCESS" = true ]; then
    SUBJECT="[SUCCESS] MongoDB Backup - $DB_NAME - $TIMESTAMP"
    BODY="<span style='color:green; font-weight:bold;'>Daily Collection-Wise MongoDB Backup And Upload To S3 Completed Successfully.</span><br><br>
$DB_INFO<br><br>
Collections Backed Up: <b>$TOTAL_COLLECTIONS/$TOTAL_COLLECTIONS</b><br>
Backup Size on S3: <b>$S3_SIZE_HUMAN</b><br>
S3 Uploaded Files: <b>$S3_OBJECTS</b><br>
Backup Duration: <b>$DURATION_HUMAN</b><br><br>"
else
    FAILED_COUNT=${#FAILED_COLLECTIONS[@]}
    SUCCESS_COUNT=$((TOTAL_COLLECTIONS - FAILED_COUNT))
    SUBJECT="[FAILED] MongoDB Backup - $DB_NAME - $TIMESTAMP"
    FAILED_LIST=$(printf '%s<br>- ' "${FAILED_COLLECTIONS[@]}")
    BODY="<span style='color:red; font-weight:bold;'>Daily Collection-Wise MongoDB Backup And Upload To S3 Encountered Errors.</span><br><br>
$DB_INFO<br><br>
Collections Backed Up: <b>$SUCCESS_COUNT/$TOTAL_COLLECTIONS</b> (Failed: $FAILED_COUNT)<br>
Partial Backup Size on S3: <b>$S3_SIZE_HUMAN</b><br>
S3 Uploaded Files: <b>$S3_OBJECTS</b><br>
Backup Duration: <b>$DURATION_HUMAN</b><br><br>
Failed Collections:<br>- ${FAILED_LIST}<br><br>
(Check full log for more details)"
fi

# Send HTML email with mailx
echo -e "$BODY" | mailx \
  -s "$SUBJECT" \
  -a "Content-Type: text/html" \
  -a "From: Mongo Backup <>" \
  -a "$LOG_FILE" \
  "$EMAIL_TO"
  ```