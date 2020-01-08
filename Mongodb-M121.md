mongo "mongodb://cluster0-shard-00-00-jxeqq.mongodb.net:27017,cluster0-shard-00-01-jxeqq.mongodb.net:27017,cluster0-shard-00-02-jxeqq.mongodb.net:27017/aggregations?replicaSet=Cluster0-shard-0" --authenticationDatabase admin --ssl -u m121 -p aggregations --norc

Key Things to remember

A $match stage may contain a $text query operator, but is must be the first stage in a pipeline.

$match should come early in an aggregation pipeline.

you cannot use $where with $match
$match uses the same query syntex as find.

{
	"_id" : ObjectId("573a1390f29313caabcd421c"),
	"title" : "A Turn of the Century Illusionist",
	"year" : 1899,
	"runtime" : 1,
	"cast" : [
		"Georges M�li�s"
	],
	"lastupdated" : "2015-08-29 00:21:21.547000000",
	"type" : "movie",
	"directors" : [
		"Georges M�li�s"
	],
	"imdb" : {
		"rating" : 6.6,
		"votes" : 580,
		"id" : 246
	},
	"countries" : [
		"France"
	],
	"genres" : [
		"Short"
	],
	"tomatoes" : {
		"viewer" : {
			"rating" : 3.8,
			"numReviews" : 32
		},
		"lastUpdated" : ISODate("2015-08-20T18:46:44Z")
	}
}

var pipeline = [{$match:{"imdb.rating":{$gte:7},"genres":{$not:{$in:['Crime','Horror']}},"rated":{$in:["PG","G"]},languages:{$all:["English","Japanese"]}}}]

Things to remember

Once we specify one field to retain, we must specify all fields we want to retain. The _id is the only exception to this.
Beyond simply removing and retaining fields, $project lets us add new fields.
$project can be used as many times as required within an Aggregation pipeline.
$project can be used to reassign values to existing field names and to derive entirely new fields.

var pipeline = [{$match:{"imdb.rating":{$gte:7},"genres":{$not:{$in:['Crime','Horror']}},"rated":{$in:["PG","G"]},languages:{$all:["English","Japanese"]}}},{$project:{_id:0,title:1,"rated":"$imdb.rating"}}]

var pipeline = [{$project:{_id:0,count:{$size:{$split:["$title"," "]}}}},{$match:{count:1}}]

{ $match: { writers: { $elemMatch: { $exists: true } } }

Optional Lab - Expressions with $project
This lab will have you work with data within arrays, a common operation.

Specifically, one of the arrays you'll work with is writers, from the movies collection.

There are times when we want to make sure that the field is an array, and that it is not empty. We can do this within $match

{ $match: { writers: { $elemMatch: { $exists: true } } }
 COPY
However, the entries within writers presents another problem. A good amount of entries in writers look something like the following, where the writer is attributed with their specific contribution

"writers" : [ "Vincenzo Cerami (story)", "Roberto Benigni (story)" ]
 COPY
But the writer also appears in the cast array as "Roberto Benigni"!

Give it a look with the following query

db.movies.findOne({title: "Life Is Beautiful"}, { _id: 0, cast: 1, writers: 1})
 COPY
This presents a problem, since comparing "Roberto Benigni" to "Roberto Benigni (story)" will definitely result in a difference.

Thankfully there is a powerful expression to help us, $map. $map lets us iterate over an array, element by element, performing some transformation on each element. The result of that transformation will be returned in the same place as the original element.

Within $map, the argument to input can be any expression as long as it resolves to an array. The argument to as is the name of the variable we want to use to refer to each element of the array when performing whatever logic we want. The field as is optional, and if omitted each element must be referred to as $$this:: The argument to in is the expression that is applied to each element of the input array, referenced with the variable name specified in as, and prepending two dollar signs:

writers: {
  $map: {
    input: "$writers",
    as: "writer",
    in: "$$writer"
 COPY
in is where the work is performed. Here, we use the $arrayElemAt expression, which takes two arguments, the array and the index of the element we want. We use the $split expression, splitting the values on " (".

If the string did not contain the pattern specified, the only modification is it is wrapped in an array, so $arrayElemAt will always work

writers: {
  $map: {
    input: "$writers",
    as: "writer",
    in: {
      $arrayElemAt: [
        {
          $split: [ "$$writer", " (" ]
        },
        0
      ]
    }
  }
}

Things to remember
The collection can have one and only one 2dsphere index.
If using 2dsphere, the distance is returned in meters. if using legacy coordinates, the distance is returned in raadians.
$geoNear must be the first stage in an aggregation pipeline.

$sort, $skip, $limit and $count are functionally equivalent to the similarly named cursor methods.
$sort can take advantages of indexes if used early within a pipeline.

By default sort will only use up to 100 megabytes of RAM. setting allowDiskUse:true will allow for larger sorts.


var pipeline = [
    { $match : { 
		"tomatoes.viewer.rating": { $gte: 3 },
		"countries": { $in: ["USA"] },
		"cast" : { $exists : true }
		} 
	},
	{ $addFields : { "num_favs" : { $size : { $setIntersection : [ "$cast", favorites ] } } } },
	{ $sort : {
	    "num_favs" : -1,
		"tomatoes.viewer.rating" : -1,
		"title" : -1
	    },
	},
	{ $skip : 24 }
];