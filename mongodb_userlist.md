```
use admin;

db.system.users.aggregate([
  {
    $unwind: "$roles"
  },
  {
    $project: {
      _id: 0,
      user: 1,
      db: 1,
      "roles__role": "$roles.role",
      "Write Access": {
        $cond: {
          if: { $in: [ "$roles.role", [ "readWrite", "dbOwner", "root" ] ] },
          then: "YES",
          else: "NO"
        }
      },
      "Super Privileges": {
        $cond: {
          if: { $eq: [ "$roles.role", "root" ] },
          then: "YES",
          else: "NO"
        }
      }
    }
  },
  { $sort: { db: 1, user: 1 } }
]);
