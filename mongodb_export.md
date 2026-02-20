```
mongoexport \
  --host <MONGODB_HOST> \
  --port <MONGODB_PORT> \
  --username <MONGODB_USERNAME> \
  --password '<MONGODB_PASSWORD>' \
  --authenticationDatabase <AUTH_DATABASE> \
  --db <DATABASE_NAME> \
  --collection <COLLECTION_NAME> \
  --out <OUTPUT_FILE>.json
```
