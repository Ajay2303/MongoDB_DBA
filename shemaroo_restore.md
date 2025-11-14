mongorestore \
  --host <HOSTNAME> \
  --port <PORT> \
  --username <USERNAME> \
  --password '<PASSWORD>' \
  --authenticationDatabase admin \
  --archive="<PATH_TO_GZ_FILE>" \
  --gzip \
  --nsFrom="<SOURCE_DB.SOURCE_COLLECTION>" \
  --nsTo="<TARGET_DB.TARGET_COLLECTION>"
