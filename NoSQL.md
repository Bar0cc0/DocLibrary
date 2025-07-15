# MongoDB

## 1. Basics Commands

```bash
python3 -m pip install pymongo

start_mongo
mongo -u root -p <password> --authenticationDatabase admin local

db.version()
show dbs
use <database>

db.createCollection("mycollection")
show collections

# insert JSON doc in "mycollection"
db.mycollection.insert({"color":"white","example":"milk"})

# nb of docs in the collection "mycollection" 
db.mycollection.count()

# list all docs in collection "mycollection"
db.mycollection.find()

exit
```

## 2. CRUD, Indexes, Aggregation Pipeline

### CREATE
```javascript
db.database.insertOne({}) // autogenerate an insertID
list_of_docs = [{doc1}, {doc2}]
db.database.insertMany(list_of_docs)
```

### READ
```javascript
db.database.findOne({key:value})
db.database.find({key:value})
db.database.find({key:value}).sort({"docID":1})
db.database.count({key:value})
```

### REPLACE
```javascript
old = db.database.findOne({key:value})
db.database.replaceOne({key:value}, old)
```

### UPDATE
```javascript
changes = {$set : {key:val, key:val}}
db.database.updateOne({key:val}, changes)
db.database.updateMany({}, changes)
```

### DELETE
```javascript
db.database.deleteOne({<docID>:<docID_value>})
db.database.deleteMany({key:value})
```

### CREATE INDEX
```javascript
db.database.createIndex({"docID":1}) // sort indexes in ascending order
```

### AGGREGATION
```javascript
db.database.aggregate([
  {$match: {key:value}},    // filter; also: $project, $sort, $count, $merge 
  {$group: {"_id":docID_value, "avgScore": {$avg:"$Score"}}}  // grouping + summary
])
```

## 3. Example 1

```javascript
start_mongo
mongo -u root -p MzI2NzItbWljaGFl --authenticationDatabase admin local
use training
db.createCollection("languages")

// insert documents
db.languages.insert({"name":"java","type":"object oriented"})
db.languages.insert({"name":"python","type":"general purpose"})
db.languages.insert({"name":"scala","type":"functional"})
db.languages.insert({"name":"c","type":"procedural"})
db.languages.insert({"name":"c++","type":"object oriented"})

// count docs in collection
db.languages.count()

// list docs in collection
db.languages.findOne()
db.languages.find()
db.languages.find().limit(3)
db.languages.find({},{"name":1}) // all docs with name only + docID in the output
db.languages.find({},{"name":0}) // all docs without name + docID in the output

// query a specific doc
db.languages.find({"name":"python"})
db.languages.find({"type":"object oriented"})
db.languages.find({"type":"object oriented"},{"name":1})

// update 
db.languages.updateMany({},{$set:{"description":"programming language"}})         // add new field to all docs 
db.languages.updateMany({"name":"python"},{$set:{"creator":"Guido van Rossum"}})  // add new field to docs where name == "python"
db.languages.updateMany({"type":"object oriented"},{$set:{"compiled":true}})      // add new field to docs where type == "object oriented" 

// delete
db.languages.remove({"name":"scala"})
db.languages.remove({"type":"object oriented"})
db.languages.remove({}) // remove all documents in the collection 

// disconnect from mongo
exit 
```

## 4. Example 2

```javascript
use training
db.createCollection("bigdata")

// insert multiple docs and time lookup function
for (i=1;i<=200000;i++){print(i);db.bigdata.insert({"account_no":i,"balance":Math.round(Math.random()*1000000)})}
db.bigdata.find({"account_no":58982}).explain("executionStats").executionStats.executionTimeMillis

// create index and time lookup function 
db.bigdata.createIndex({"account_no":1})
db.bigdata.getIndexes()
db.bigdata.find({"account_no": 69271}).explain("executionStats").executionStats.executionTimeMillis

// delete indexes 
db.bigdata.dropIndex({"account_no":1})
```

## 5. Example 3

```javascript
db.marks.insert({"name":"Ramesh","subject":"maths","marks":87})
db.marks.insert({"name":"Ramesh","subject":"english","marks":59})
db.marks.insert({"name":"Ramesh","subject":"science","marks":77})
// ...

// limit the number of documents printed in the output
db.marks.aggregate([{"$limit":2}])

// sort the documents based on field "marks" in ascending order
db.marks.aggregate([{"$sort":{"marks":1}}])

// idem in descending order
db.marks.aggregate([{"$sort":{"marks":-1}}])

// simple pipeline 
db.marks.aggregate([
  {"$sort":{"marks":-1}},
  {"$limit":2}
])

// more complex pipeline equiv. to: > SELECT subject, avg(marks) FROM marks GROUP BY subject;
db.marks.aggregate([
{
    "$group":{
        "_id":"$subject",
        "average":{"$avg":"$marks"}
        }
}
])

// add other stages to previous pipeline: "find the top 2 students by avg marks"
db.marks.aggregate([
  {"$group":{"_id":"$name",
             "average":{"$avg":"$marks"}}},
  {"$sort":{"average":-1}},
  {"$limit":2}
])
```

# Cassandra

## 1. Basic Commands

```bash
start_cassandra
cqlsh --username cassandra --password MjEwMzUtbWljaGFl
show host
show version
describe keyspaces
cls 
exit 
```

## 2. Shards, Tables, CRUD

### KEYSPACE 
```sql
CREATE KEYSPACE training WITH replication = {
    'class':'SimpleStrategy', 
    'replication_factor' : 3
};

DESCRIBE KEYSPACES 
DESCRIBE training

ALTER KEYSPACE training
WITH replication = {'class': 'NetworkTopologyStrategy'};

USE training;
DESCRIBE TABLES

DROP KEYSPACE training;
USE system;
DESCRIBE KEYSPACES
```

### TABLE
```sql
USE training;
CREATE TABLE movies(
    movie_id int PRIMARY KEY,
    movie_name text,
    year_of_release int
);

ALTER TABLE movies ADD genre text;

DROP TABLE movies;
```

### INSERT
```sql
INSERT into movies(movie_id, movie_name, year_of_release)
    VALUES (1,'Toy Story',1995);
INSERT into movies(movie_id, movie_name, year_of_release)
    VALUES (2,'Jumanji',1995);
INSERT into movies(movie_id, movie_name, year_of_release)
    VALUES (3,'Heat',1995);
INSERT into movies(movie_id, movie_name, year_of_release)
    VALUES (4,'Scream',1995);
INSERT into movies(movie_id, movie_name, year_of_release)
    VALUES (5,'Fargo',1996);
```

### RUD (Read, Update, Delete)
```sql
SELECT * FROM movies;
SELECT movie_name FROM movies WHERE movie_id = 1;

UPDATE movies SET year_of_release = 1996 WHERE movie_id = 4;

DELETE from movies WHERE movie_id = 5;
```

### INDEX
```sql
CREATE INDEX IF NOT EXISTS index_name
    ON keyspace_name.table_name ( KEYS ( column_name ) )
```

### cqlsh CLI
```bash
cqlsh [options] [host [port]] 
```

Options:
- `--help`, `--version`
- `-u --user`, `-p --password`
- `-k --keyspace`, `-f --file`
- `--request-timeout`

Special commands: 
- `CAPTURE`, `CONSISTENCY`, `COPY`
- `DESCRIBE`, `EXIT`, `PAGING`, `TRACING`

# Cloudant

## 1. Dashboard: Create Database

Replication config:

Under Source:
- Select Type = Local database
- Select Name = training
- Select Authentication = "IAM Authentication"
- Paste the api key you copied earlier in the IAM API Key textbox.
 
Under Target:
- Select Type = New local database
- Select Name = training_replica
- Select Authentication = "IAM Authentication"
- Paste the api key you copied earlier in the IAM API Key textbox.
 
Under Options:
- Select Type = Continuous

## 2. Query

WHERE equivalent:
```json
{
   "selector": {}
}
```

Query example 1:
```json
{
   "selector": {
      "_id": {
         "$gt": "4" 
      }
   }
}
```
Note: `$gt` means greater than, also available: `$lt` for less than

Query example 2:
```json
{
   "selector": {
   "_id": {
         "$lt": "4"
      }
   },
   "fields": [
      "_id",
      "price",
      "square_feet"
   ]
}
```

Query example 3:
```json
{
   "selector": {
      "_id": {
         "$gt": "2"
      }
   },
   "fields": [
      "_id",
      "price",
      "square_feet"
   ],
   "sort": [
      {
         "_id": "asc"
      }
   ]
}
```

## 3. CURL Basics

### Syntax
```bash
# retrieval
curl <url of webpage>

# create variable
export VAR="https://username:password@host"

# create alias
alias acurl="curl -sgH 'Content-type: application/json'"

# test connectivity
acurl $VAR/

# retrieve db list
acurl $VAR/_all_dbs

# add formatting
acurl $VAR/_all_dbs | python -m json.tool 

# retrieve specific db
acurl $VAR/training 

# view docs content
acurl $VAR/training/_all_docs?include_docs=true   # use this to retrieve _rev token values

# single doc content
acurl $VAR/training/<docID>
```

### HTTP Methods
```bash
# create db
curl -X PUT $VAR/training 

# drop db
curl -X DELETE $VAR/training 

# insert doc
curl -X PUT $VAR/training/"docID" -d '{key:values}'

# update doc
curl -X PUT $VAR/training/"docID" -d '{key:values, ..., _rev:"token value"}'  # must provide all fields of a doc for request to be successful

# delete doc
curl -X DELETE $VAR/training/docID?rev=token_value
```

### QUERY
```bash
# query syntax
curl -X POST $VAR/training/_find \
      -H "Content-Type: application/json" \
      -d '{"selector":{"_id":"docID"}}' 

# complex or frequent queries can be stored in a JSON file and called using '-d @jsonfilename'
curl -X POST $VAR/training/_find -d @myQuery.json
```

## 4. Indexes

Create JSON index:
```bash
curl -X POST $SERVERURL/employees/_index \
     -H"Content-Type: application/json" \
     -d'{
        "index": {
            "fields": ["firstname"]
        }
     }'
```

Create JSON index with name:
```bash
curl -X POST $SERVERURL/employees/_index \
     -H"Content-Type: application/json" \
     -d'{
        "index": {
            "fields": ["firstname"]
        },
        "name" : "firstname-index",
        "type" : "json"
     }'
```

Create text index:
```bash
curl -X POST $SERVERURL/employees/_index \
     -H"Content-Type: application/json" \
     -d'{ "index": {},
        "type": "text"
     }'  
```

## 5. Example: Querying Data with HTTP API

```bash
export CLOUDANTURL="https://apikey-v2-1ktn8d6fuuo6kjoz145fris5ccx24fhmirsku7o3q7bh:6b3dc2399426437b2397e08ac9cf0184@4646e655-6aee-42d8-8b93-d2bde6e9a6ca-bluemix.cloudantnosqldb.appdomain.cloud"
curl $CLOUDANTURL
curl $CLOUDANTURL/_all_dbs

curl -X PUT $CLOUDANTURL/animals
curl -X DELETE $CLOUDANTURL/animals

curl -X PUT $CLOUDANTURL/planets
curl -X PUT $CLOUDANTURL/planets/"1" -d '{ 
    "name" : "Mercury" ,
    "position_from_sum" :1 
     }'
curl -X GET $CLOUDANTURL/planets/1 

curl -X PUT $CLOUDANTURL/planets/1 -d '{ 
    "name" : "Mercury" ,
    "position_from_sum" :1,
    "revolution_time":"88 days",
    "_rev":"1-3fb3ccfe80573e1ae334f0cfa7304f6c"
    }'

curl -X PUT $CLOUDANTURL/planets/1 -d '{
    "name": "Mercury",
    "position_from_sum": 1,
    "revolution_time": "88 days",
    "rotation_time": "59 days",
    "_rev": "1-3fb3ccfe80573e1ae334f0cfa7304f6c"
    }'

curl -X DELETE $CLOUDANTURL/planets/1?rev=2-de9fdd2d971e377c5db2d6425cb38ff1

curl -X PUT $CLOUDANTURL/diamonds

curl -X PUT $CLOUDANTURL/diamonds/1 -d '{
    "carat": 0.31,
    "cut": "Ideal",
    "color": "J",
    "clarity": "SI2",
    "depth": 62.2,
    "table": 54,
    "price": 339
  }'
curl -X PUT $CLOUDANTURL/diamonds/2 -d '{
    "carat": 0.2,
    "cut": "Premium",
    "color": "E",
    "clarity": "SI2",
    "depth": 60.2,
    "table": 62,
    "price": 351
  
  }'
curl -X PUT $CLOUDANTURL/diamonds/3 -d '{
    "carat": 0.32,
    "cut": "Premium",
    "color": "E",
    "clarity": "I1",
    "depth": 60.9,
    "table": 58,
    "price": 342

  }'
curl -X PUT $CLOUDANTURL/diamonds/4 -d '{
    "carat": 0.3,
    "cut": "Good",
    "color": "J",
    "clarity": "SI1",
    "depth": 63.4,
    "table": 54,
    "price": 349
  }'
curl -X PUT $CLOUDANTURL/diamonds/5 -d '{
    "carat": 0.3,
    "cut": "Good",
    "color": "J",
    "clarity": "SI1",
    "depth": 63.8,
    "table": 56,
    "price": 347

  }'  

curl -X POST $CLOUDANTURL/diamonds/_find \
-H"Content-Type: application/json" \
-d'{ 
    "selector":
        {
            "_id":"1"
        }
    }'
curl -X POST $CLOUDANTURL/diamonds/_find \
-H"Content-Type: application/json" \
-d'{ "selector":
        {
            "carat":0.3
        }
    }'
curl -X POST $CLOUDANTURL/diamonds/_find \
-H"Content-Type: application/json" \
-d'{ "selector":
        {
            "price":
                {
                    "$gt":345
                }
        }
    }'

curl -X POST $CLOUDANTURL/diamonds/_index \
-H"Content-Type: application/json" \
-d'{
    "index": {
        "fields": ["price"]
    }
}'

curl -X POST $CLOUDANTURL/diamonds/_find \
-H"Content-Type: application/json" \
-d'{ "selector":
        {
            "price":
                {
                    "$gt":345
                }
        }
    }'
```