### MongoDB query/script that prints collection stats in this format (collection name, document count, data size, storage size, index size).

```
var dbObj = db.getSiblingDB("your_db_name");

dbObj.getCollectionNames().forEach(function(collName) {

    var stats = dbObj.getCollection(collName).stats();

    print("Collection: " + collName);
    print("Document count: " + stats.count);
    print("Data size (MB): " + (stats.size / (1024 * 1024)).toFixed(2));
    print("Storage size (MB): " + (stats.storageSize / (1024 * 1024)).toFixed(2));
    print("Index size (MB): " + (stats.totalIndexSize / (1024 * 1024)).toFixed(2));
    print("----------------------------------------");

});
```
