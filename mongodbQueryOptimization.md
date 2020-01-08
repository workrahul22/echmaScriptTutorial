#	Mongodb Query Optimization

## Create Indexes to Support Queries

For commonly issued queries, create indexes. If a query searches multiple fields, create a compound index. Scanning an index is much faster than scanning a collection.

If you have a posts collection containing blog posts, and if you regularly issue a query that sorts on the author_name field, then you can optimize the query by creating an index on the author_name field.

> db.posts.createIndex({ author_name : 1})

For single-field indexes, the selection between ascending and descending order is immaterial. For compound indexes, the selection is important.

The db.collection.createIndex method only creates an index of the same specification does not already exist.

Mongodb indexes use a B-tree data structure.

You can create indexes with custom name.

> db.products.createIndex(
>	{ item:1, quantity: -1},
>	{name:"query for inventory"}
>)

you can view index names using the db.collection.getIndexes() method. You cannot rename an index once created. Instead, you must drop and re-create the index with a new name.

### Single Index:

> db.users.createIndex({ email: 1})

If your collection have the object field like an address that store the information like city, state and country then you add the index like below.

> db.users.createIndex( {"address.city" : 1})

### Compound Index:

This index are always created with minimum two fields from the collection. For example, below index was created with ascending order on fullName and userName field.

> db.users.createIndex({fullName:1,userName:1})

Mongodb have to limit for compound index only on 31 fields.
The order of the fields listed in a compound index is important.

### Text Index:

If you need to search for text or array fields, then add text index. Text indexes can include any field whose value is a string or an array of string elements.

> db.users.createIndex( {comments: "text" })

### Unique Single Index

A unique index ensures that the indexed fields do not store duplicate values.

> db.user.createIndex({ "userId": 1}, {unique:true})


### Unique Compound Index:

You use the unique constraint on compund index, then MongoDb will enforce uniqueness on the combination of the index key values.

Indexes also improve efficiency on queries that routinely sort on a given field.

If you regularly issue a query that sorts on the timestamp field, then you can optimize the query by creating an index on the timestamp field.

> db.posts.createIndex({ timestamp: 1 })
> db.posts.find().sort({ timestamp: -1 })

Because Mongodb can read indexes in both ascending and descending order, the direction of a single-key index does not matter.

Indexes support queries, update operations, and some phases of aggregation pipeline.

## Limit the Number of Query Results to Reduce Network Demand.

MongoDB cursors return results in groups of multiple documents. If you know the number of results you want, you can reduce the demand on network resources by issuing the limit() method.

> db.posts.find().sort({ timestamp : -1 }).limit(10)

## Use Projection to Return Only Necessary Data.

When you need only a subset of fields from documents, you can achieve better performance by returning only the fields you need.

> db.posts.find({},{ timestamp:1, titile:1, author:1, abstract:1}).sort({timestamp:-1})

## Use $hint to select a Particular Index

In most cases the query optimizer selects the optimal index for a specific operation;  however you can force MongoDb to use a specific index using the hint() method. Use hint() to support performance testing, or on some queried you must select a field or field included in several indexes.

## Use the increment operator to perform operations server-side.

Use MongoDb's $inc operator to increment or decrement values in documents. The operator increments the value of the field on the server side, as an alternative to selecting a document, making simple modifications in the client and then writing the entire document to the server. The $inc operator can also help avoid race conditions, which would result when two application instances queried for a document, manually incremented a field, and saved the entire document black at the same time.

## Aggregation Pipeline Optimization

### Projection Optimization

Add only require fields from the collection and reducing the amount of data passing through the pipeline.

Example

> $project
> {
	> fullName:1,
	> email:1,
	> address: 0	
> }

### Pipeline Sequence Optimization

Always maintain the sequence like $match + $projection, $match1 + $projection1 ad so on in queue. The sequence has reduced the query execution time because data are filtered before going to projection.

### $match and $sort

Define index to match and sort field because it uses the index technique. Always add $match and $sort on an aggregation first stage if possible.

### $sort and $limit

Use $sort before $limit in the aggregate pipeline like $sort+$limit+$skip. $sort oprator can take advantage of an index when placed at the beginning of the pipeline or placed before the $project, $unwind, and $group aggregation operators.

The $sort stage has a limit of 100 megabytes of RAM, so use allowDiskUse option true to not consume too much RAM.

### $skip and $limit:

Always use $limit before $skip in aggregation pipeline.

### $lookup and $unwind

Always create an index on the foreignField attributes in $lookup, unless the collection are trivial size.
If $unwind follows immediately after $lookup, then use $unwind in $lookup.

>{
>	$lookup: {
>		from: "otherCollection",
>		as: "resultingArrays",
>		localField: 'x',
>		foreignField: "y",
>		unwinding: {preserveNullAndEmptyArrays: false}
>	}
>}

### AllowDiskUse in aggregate

AllowDiskUse: true, aggregation operations can write data to the _tmp subdirectory in the Database Path directory. It is used to perform the large query on temp directory.

> db.orders.aggregate(
>	[
>		{$match:{status:"A"}},
>		{$group:{_id:"$cust_id", total:{$sum:"$amount"}}},
>		{$sort:{tatal:-1}}
>	],
>	{
>		allowDiskUse: true
>	}
>)

## Rebuild the index on collection:

Index rebuild is required when you add and remove indexes on fields at multiple times on collection.

> db.user.reIndex();

This operation drops all indexes for a collection, including the _id index, and then rebuilds all indexes.


### Remove Too Many index:

Add only required index on the collection because the index will consume the CPU for write operation.

If compound index exist then remove single index from collection fields.

## Analyze Query Performance:

To analyze the query performance, we can check the query execution time, no of records scanned and much more.

The explain() method returns a document with the query plan and, optionally, the execution statistics.

The explain() Method used the three different options for returning the execution information.

The possible options are "queryPlanner", "executionStats" and "allPlansExecution" and queryPlanner is default.

> db.users.find({sEmail:'demo@test.com'}).explain()
> db.users.find({sMobile:'9685741425'}).explain("executionStats")

Main point that we have to take care on above explanation:

* queryPlanner.winningPlan.stage: displays COLLSCAN to indicate a collection scan. This is a generally expensive operation and can result in slow queries.

* executionStats.nReturned : displays 3 to indicate that the query matches and returns three documents.

* executionStats.totalKeysExamined: displays 0 to indicate that this query is not using an index.

* executionStats.totalDocsExamined: displays 10 to indicate that MongoDB had to scan ten documents(i.e all documents in the collection) to find the three matching documents.

* queryPlanner.winningPlan.inputStage.stage: display IXSCAN to indicate index use.

The explain() method can be used in many ways like below.

>db.orders.aggregate(
>    [
>        { $match: { status: "A" } },
>        { $group: { _id: "$custId", total: { $sum: "$amount" } } },
>        { $sort: { total: -1 } }
>    ],
>    {explain: true}
>);

>db.orders.explain("executionStats").aggregate(
>    [
>        {$match: {status: "A", amount: {$gt: 300}}}
>    ]
>);

>db.orders.explain("executionStats").aggregate(
>    [
>        {$match: {status: "A", amount: {$gt: 300}}}
>    ]
>);

## Check Your Mongodb log

By default, Mongodb records all queries which take longer than 100 milliseconds. Its location is defined in your configuration's systemLog.path setting, and it's normally /var/log/mongodb/mongod.log in Debian based distribution such as ubuntu.

The log file can be large, so you may want to clear it before profiling. From the mongo command-line console, enter.

> use admin;
> db.runCommand({logRotate:1});

A new log file will be started and the old data will be available in file named with the backup date and time. You can delete the backup or move it elsewhere for further analysis.

## Create Two or more connection objects.

Performance can be improved by defining more than one database connection object. For example

1. one to handle the majority of fast queries
2. one to handle slower document inserts and updates.
3. one to handle complex report generation.

Each object is treated as a separate database client and will not delay the processing of others. The application should remain responsive.

## Set Maximum Execution Times

MongoDb commands run as long as they need. A slowly executing query can hold up others, and your web application may eventually time out. This can throw various strange instability problems in Nodejs programs, which happily continue to wait for an asynchronous callback.

You can specify a time limit in milliseconds using maxTimeMS() 

> db.user.find({city:/^A.+/i}).maxTimeMS(100);

You should set a reasonable maxTimeMS value for any command which is likely to take considerable time. Unfortunately, Mongodb doesn't allow you to define a global timeout value, and it must be set for individual queries ( although some libraries may apply a default automatically).
