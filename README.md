# Mongo Rocks

## Prerequisites
You'll need to have the following ready to go:

1. MongoDB (3.6 or above) -  Get is from the [MongoDB Download Centre](https://www.mongodb.com/download-center?community#community).
2. A GUI - There are a few, my favourite is [RoboMongo](https://robomongo.org/download).

## Getting started
### Running MongoDB
Mongo will install (by default) in:

   C:\Program Files\MongoDB\Server\<Version>\bin

Navigating to this you'll see all the interesting files for working with Mongo.

* `bsondump.exe`
* `mongo.exe`
* `mongod.exe`
* `mongod.pdb`
* `mongodump.exe`
* `mongoexport.exe`
* `mongofiles.exe`
* `mongoimport.exe`
* `mongoperf.exe`
* `mongorestore.exe`
* `mongos.exe`
* `mongos.pdb`
* `mongostat.exe`
* `mongotop.exe`

`mongod.exe` is the Mongo server, we can run it either as a service (as it would be in production) or as a console application (which is easier when you're playing about with it).  

By default it will:

* Listen for local connections on port `27027`.
* Store files in `C:\data\db\`
* Run without any authentication.
* Will use the WiredTiger database engine.

`mongod.exe` will accept arguments to change these and other settings, `mongod.exe --help` will list them all out.  We're not going to use the default file storage location, when trying stuff out it's good to have it somewhere else so you can simply delete the directory when you're finished without affecting anything else.

Create `C:\MongoRocksData`.  Now run `mongod.exe --dbpath C:\MongoRocksData`.  You'll see a wealth of information including warnings about running in a non-production-safe way:

    ** WARNING: Access control is not enabled for the database.
    **          Read and write access to data and configuration is unrestricted.
    ** WARNING: This server is bound to localhost.
    **          Remote systems will be unable to connect to this server.
    **          Start the server with --bind_ip <address> to specify which IP
    **          addresses it should serve responses from, or with --bind_ip_all to
    **          bind to all interfaces. If this behavior is desired, start the
    **          server with --bind_ip 127.0.0.1 to disable this warning.

Soon afterwards you should see that Mongo is up and listening for connections:

    createCollection: admin.system.version with provided UUID: d6efacd2-c7cb-4ca7-b1ca-03d9ef8ad0ae
    setting featureCompatibilityVersion to 3.6
    createCollection: local.startup_log with generated UUID: d061fc00-0b2a-46b1-b86c-9e447f25c98d
    Initializing full-time diagnostic data capture with directory 'C:/MongoRocksData/diagnostic.data'
    waiting for connections on port 27017

Now we're ready to connect.

## Connecting to a MongoDB Server

`mongo.exe` provides us with a CLI to work with Mongo (there are drivers for languages and GUIs available as well, we'll work with the CLI for a bit though).  By default `mongo.exe` will:

* Try to connect at `localhost:27027`.
* Will not authenticate.

So, we want the defaults so just run `mongo.exe` then we'll firstly see more logs appear on the server's output:

    [listener] connection accepted from 127.0.0.1:59218 #1 (1 connection now open)
    [conn1] received client metadata from 127.0.0.1:59218 conn: 
    { 
        application: { 
            name: "MongoDB Shell" 
        }, 
        driver: { 
            name: "MongoDB Internal Client", 
            version: "3.6.0" 
        }, os: { 
            type: "Windows", 
            name: "Microsoft Windows 10", 
            architecture: "x86_64", 
            version: "10.0 (build 14393)" 
        } 
    }
    
> NB:  I've formatted the output so that it's readable.

We'll also see that the client has written some warning about this not being a recommended configuration as well as the command prompt change to indicate that its waiting for input.

## CRUD
### Databases & Collections

Data in Mongo is contained _collections_ which are in _databases_.  There can be many databases on any server and each database can contain many collections.

Firstly let's see what databases exist by default.

    > show dbs
    admin   0.000GB
    config  0.000GB
    local   0.000GB

These all have roles to play but are not used to store you documents, in fact if you were to put them in any of these databases than they may well get deleted or the server might just break.

Creating databases and collections is done implicitly.  If you try to write to a database and\or collection that doesn't exist then it'll just get created.

When we're working in the shell we'll always to working in the context of a single database.  The reference to that database is stored in the `db` variable.  Just type `db` to see what it's pointing to:

    > db
    test

When the Mongo client first connects to a server it will always be in the context of the `test` database.  `test` doesn't exist, but that doesn't matter because it will be created when it's needed.

To change the database context you're working in (i.e. the value of `db`) use the `use <database_name>` command.

    > use mongo_rocks
    switched to db mongo_rocks

We can see what collections are in a database with `show collections`.  There aren't any yet, there will be soon.

### BSON Documents

BSON is the format that Mongo stores and retrieved data in.  Based on JSON, BSON is is nested structure that contains key/value pairs of objects, arrays and data types.  Unlike JSON, BSON is not a text file.  BSON also supports more data types than JSON, these are described in the [MongoDB documentation](https://docs.mongodb.com/manual/reference/bson-types/) and in the [BSON Specification](http://bsonspec.org/spec.html).  In BSON language, the data structure is referred to as a _document_, as are all objects in the document (_nested documents_).

> JSON is not completely a subset of BSON as BSON doesn't allow `0x00` character.  BSON reserves `0x00` for control purposes such as string termination.


Take this BSON document:

    
    {
        "_id" : ObjectId("0123456789abcdef01234567"),
        "name" : "David"
    }

`ObjectId` is not valid JSON.  This is a BSON object similar to a GUID, but optimised for generation speed over guaranteed uniqueness.  If a document is created without an `_id` field then Mongo will create one and assign it a new `ObjectId`.  

The BSON file created for this would have this structure:

| Description | Bytes | Value |
|-------------|-------|-------|
| File size | `26` `00` `00` `00` | `38` |
| Data type | `07` | _ObjectId_ |
| Field name | `5F` `69` `64` | `_id` |
| String terminator | `00` |  |
| Field Value | `01` `23` `45` `67` `89` `AB` `CD` `EF` `01` `23` `45` `67` | `0123456789ABCDEF01234567` |
| Data type | `02` | _String_ |
| Field name | `6E` `61` `6D` `65` | `name` |
| String terminator | `00` |  |
| Field Length | `06` `00` `00` `00` | `6` |
| Field value | `44` `61` `76` `69` `64` | `David` |
| String terminator | `00` |  |
| Object terminator | `00` |  |

> NB: There is a 64Mb limit on any document size.


Mongo provides tooling to export the document to JSON, when it does it used some JSON compliant structures to capture data types that are not JSON compliant.

The above document would be serialised to JSON like so:

    {
        "_id": {
            "$oid":"0123456789abcdef01234567"
        },
        "name":"David"
    }

### Inserting Documents

Let's insert a document into a collection called `names` in a database called `mongo_rocks`:

    > use mongo_rocks
    switched to db mongo_rocks
    > db.names.insert({
    ... _id: NumberInt(1),
    ... name: 'David'
    ... })
    WriteResult({ "nInserted" : 1 })

Every document needs a unique ID, this is always the `_id` field.  If you don't set it explicitly then Mongo will create a new `ObjectId` and assign its value to it.

We can also add an array of documents (as long as the total doesn't exceed 64 Mb):

    > db.names.insertMany([{
    ... _id: NumberInt(2),
    ... name: 'Alice'
    ... }, {
    ... _id: NumberInt(3),
    ... name: 'Bob'
    ... }, {
    ... _id: NumberInt(4),
    ... name: 'Charlie'
    ... }])
    {
            "acknowledged" : true,
            "insertedIds" : [
                    NumberInt(2),
                    NumberInt(3),
                    NumberInt(4)
            ]
    }

When we add many, we get the `_id` fields returned to us.  If we try to insert a duplicate `_id` then we get an exception:

    > db.names.insert({
    ... _id: NumberInt(2),
    ... name: 'Eve'
    ... })
    WriteResult({
            "nInserted" : 0,
            "writeError" : {
                    "code" : 11000,
                    "errmsg" : "E11000 duplicate key error collection: test.names index: _id_ dup key: { : 2 }"
            }
    })

### Retrieving Documents
We use the find method to retrieve documents, or rather we use the cursor returned by the find method.  The method has the syntax:

    find(<filter>, <projection>)

Both the filter and the projection are optional.

To retrieve all the documents we just call the find with either no arguments, or with an empty object.

    > db.names.find()
    { "_id" : 1, "name" : "David" }
    { "_id" : 2, "name" : "Alice" }
    { "_id" : 3, "name" : "Bob" }
    { "_id" : 4, "name" : "Charlie" }

The reason that we're seeing the documents, rather than seeing a cursor is because the client is figuring out that we probably want to get the documents, therefore it's iterating the documents with the cursor and displaying them without you having to tell it too.  We'll see later that this doesn't happen when there are a lot of documents returned.

The projection object tells the server what fields we want to see.  We have the option to either give it a white-list or a black-list (not both).  For example, if we only wanted to see the `_id` field then we'd pass:

    > db.names.find({}, {
    ... _id: 1
    ... })
    { "_id" : 1 }
    { "_id" : 2 }
    { "_id" : 3 }
    { "_id" : 4 }

If we wanted to exclude the `_id` field then we'd pass a `0` instead:

    > db.names.find({}, {
    ... _id: 0
    ... })
    { "name" : "David" }
    { "name" : "Alice" }
    { "name" : "Bob" }
    { "name" : "Charlie" }

There is a _gotcha_ here,  The `_id` field is always returned unless you explicitly exclude it.  So to get only the names we need to pass both the `name` field and the `_id` field to the projection:

    > db.names.find({}, {
    ... _id: 0,
    ... name: 1
    ... })
    { "name" : "David" }
    { "name" : "Alice" }
    { "name" : "Bob" }
    { "name" : "Charlie" }
    
>  NB:  This is the only example of `1` and `0` appearing in the projection.

We can search for specific documents by providing a filter document:

    > db.names.find({
    ... name: 'David'
    ... }
    ... )
    { "_id" : 1, "name" : "David" }

### Removing Documents

Removing documents is similar to finding them.  We pass in a the same filter object as we would when finding and also an options object.  There is a is safety feature though, the filter object isn't optional (to prevent you from the accidental pressing of go and deleting the entire database).

    db.names.remove(<filter>)
    
### Updating Documents

Updating documents requires two arguments.  Firstly a filter to determine which documents the update affects and secondly a document describing what the update to perform is.

> NB: By default update will only affect the first document, we have to tell it if we don't want this constraint.

If we wanted to add a date of birth to Alice we could write:

    > db.names.update({
    ... name: 'Alice'
    ... }, {
    ... $set: {
    ... date_of_birth: ISODate('30-01-1995')
    ... }
    ... })
    WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
    
Notice the use of the `$set` operator field.  This tells Mongo that you want to insert a new field, without it Mongo would think that you wanted to replace the whole document.

    > db.names.find({
    ... name: 'Alice'
    ... })
    { "_id" : 2, "name" : "Alice", "date_of_birth" : ISODate("1995-01-30T00:00:00Z") }
    

### Scripting

The Mongo client will execute arbitrary JavaScript.  For example, we could write a loop to add 1000 documents:
    
    > for(var i = 0; i < 1000; i++) {
    ... db.names.insert({number: NumberInt(i)});
    ... }
    WriteResult({ "nInserted" : 1 })
    
When we search now we see lots of documents:

    > db.names.find({}, {_id: 0})
    { "number" : 0 }
    { "number" : 1 }
    { "number" : 2 }
    { "number" : 3 }
    { "number" : 4 }
    { "number" : 5 }
    { "number" : 6 }
    { "number" : 7 }
    { "number" : 8 }
    { "number" : 9 }
    { "number" : 10 }
    { "number" : 11 }
    { "number" : 12 }
    { "number" : 13 }
    { "number" : 14 }
    { "number" : 15 }
    { "number" : 16 }
    { "number" : 17 }
    { "number" : 18 }
    { "number" : 19 }
    Type "it" for more

Now we have a good number of documents we can see the batching behaviour of the cursor, only twenty documents are returned.  We iterate the cursor with the `it` command.
    
    > it
    { "number" : 20 }
    { "number" : 21 }
    { "number" : 22 }
    { "number" : 23 }
    { "number" : 24 }
    { "number" : 25 }
    { "number" : 26 }
    { "number" : 27 }
    { "number" : 28 }
    { "number" : 29 }
    { "number" : 30 }
    { "number" : 31 }
    { "number" : 32 }
    { "number" : 33 }
    { "number" : 34 }
    { "number" : 35 }
    { "number" : 36 }
    { "number" : 37 }
    { "number" : 38 }
    { "number" : 39 }
    Type "it" for more

We can also load scripts into the shell.  If we have the following `hello.js` file:

    function Hello(name) {
        print('Hello ' + name + '!');
    }

We can load it into the shell:

    > load('.../hello-world.js')
    true
    
Now we can call the function:

    > Hello('Dave')
    Hello Dave!
    
JavaScript is powerful and might be an unacceptable attack vector.  You can prevent JavaScript from being executed by passing the `--noscripting` option when you start `mongod.exe`.  Scripting on the client is never prevented.

### Importing and Exporting

Mongo has two exporting tools and two importing tools, to be used in pairs:

* `mongoexport.exe` & `mongoimport.exe`
* `mongodump.exe` & `mongorestore.exe`

The first two work with JSON, the next two work with BSON and meta-data.  For a full backup/restore you need to use the second pair.  Not only will it include the data in native BSON but it will also include the indexes etc.

    ...>mongoexport --collection names --out .\names-export.json
    2018-01-16T15:49:21.203+0000    connected to: localhost
    2018-01-16T15:49:21.254+0000    exported 1000 records

This gives you a file like: 

    {"_id":{"$oid":"5a5e1554c469f5667ecde31c"},"number":0}
    {"_id":{"$oid":"5a5e1554c469f5667ecde31d"},"number":1}
    {"_id":{"$oid":"5a5e1554c469f5667ecde31e"},"number":2}
    ...
    {"_id":{"$oid":"5a5e1554c469f5667ecde703"},"number":998}
    {"_id":{"$oid":"5a5e1554c469f5667ecde703"},"number":999}
    
 If we dump the collection instead of exporting it we'd use:
    
    ...>mongodump --db mongo_rocks --collection names --out .\names-export
    2018-01-16T15:52:58.138+0000    writing mongo_rocks.names to
    2018-01-16T15:52:58.151+0000    done dumping mongo_rocks.names (1 document)


This gives us a directory with two files in it:
    
    ...>tree names-export /a /f
    Folder PATH listing for volume OS
    Volume serial number is DEB6-B9B1
    C:\USERS\D.BANKS\DOCUMENTS\DEVELOPMENT\NAMES-EXPORT
    \---mongo_rocks
            names.bson
            names.metadata.json

The `names.bson` file contains all out data in BSON format.  The `names.metadata.json` file contains information about the collection so that it can be rebuilt accurately.  This on contains:

    {
    	"options": {},
    	"indexes": [{
    		"v": 2,
    		"key": {
    			"_id": 1
    		},
    		"name": "_id_",
    		"ns": "mongo_rocks.names"
    	}],
    	"uuid": "dc47a6aa6a0c4c368688c1cc7fa5775f"
    }
    
We can see that there is a single index in this database.  It's called `_id_` and it's on the `_id` field ascending (descending would be a `-1` instead).

We've going to clear the database and restore one that's already made.  In the shell, make sure you're in the `mongo_rocks` database and issue this command:

    > show dbs
    admin        0.000GB
    config       0.000GB
    local        0.000GB
    mongo_rocks  0.002GB
    > use mongo_rocks
    switched to db mongo_rocks
    > db.dropDatabase()
    > show dbs
    admin   0.000GB
    config  0.000GB
    local   0.000GB

Now restore from the `mongo-rocks-dump` directory (download it compressed from [here](https://bitbucket.org/BanksySan/mongorocks/downloads/mongo-rocks-dump.v1.zip) and uncompress it). 


    ...> mongorestore --nsInclude mongo_rocks.students mongo_rocks\mongo-rocks-dump
    preparing collections to restore from
    reading metadata for mongo_rocks.students from mongo_rocks\mongo-rocks-dump\mongo_rocks\students.metadata.json
    restoring mongo_rocks.students from mongo_rocks\mongo-rocks-dump\mongo_rocks\students.bson
    no indexes to restore
    finished restoring mongo_rocks.students (10000 documents)
    done
    
Now, for the last time, open the Mongo shell again.  Have a look around and satisfy yourself that it's in good order:

    > show dbs
    admin        0.000GB
    config       0.000GB
    local        0.000GB
    mongo_rocks  0.002GB
    > use mongo_rocks
    switched to db mongo_rocks
    > show collections
    students
    > db.students.find().count()
    10000
    > db.students.find().limit(1).pretty()
    {
            "_id" : 1,
            "personal" : {
                    "gender" : "Male",
                    "given_names" : [
                            "Trent",
                            "Paul"
                    ],
                    "surname" : "James",
                    "date_of_birth" : ISODate("1984-04-28T23:00:00Z")
            },
            "enrollments" : [
                    {
                            "start_date" : ISODate("2003-01-29T00:00:00Z"),
                            "name" : "Psychology",
                            "school" : "Humanities",
                            "score" : 27
                    },
                    {
                            "start_date" : ISODate("2003-01-29T00:00:00Z"),
                            "name" : "Malay",
                            "school" : "Languages",
                            "score" : 46
                    },
                    {
                            "start_date" : ISODate("2003-01-29T00:00:00Z"),
                            "name" : "Product Design",
                            "school" : "Technology",
                            "score" : 46
                    },
                    {
                            "start_date" : ISODate("2003-01-29T00:00:00Z"),
                            "name" : "Social Science",
                            "school" : "Humanities"
                    },
                    {
                            "start_date" : ISODate("2003-01-29T00:00:00Z"),
                            "name" : "Law",
                            "school" : "Humanities",
                            "score" : 67
                    },
                    {
                            "start_date" : ISODate("2004-01-29T00:00:00Z"),
                            "name" : "History",
                            "school" : "Humanities",
                            "score" : 84
                    },
                    {
                            "start_date" : ISODate("2004-01-29T00:00:00Z"),
                            "name" : "Home Economics: Food and Nutrition",
                            "school" : "Humanities"
                    },
                    {
                            "start_date" : ISODate("2004-01-29T00:00:00Z"),
                            "name" : "Classical Greek",
                            "school" : "Languages",
                            "score" : 93
                    },
                    {
                            "start_date" : ISODate("2004-01-29T00:00:00Z"),
                            "name" : "Electronics with Resistant Materials",
                            "school" : "Technology",
                            "score" : 54
                    },
                    {
                            "start_date" : ISODate("2005-01-29T00:00:00Z"),
                            "name" : "Gujarati",
                            "school" : "Languages",
                            "score" : 45
                    },
                    {
                            "start_date" : ISODate("2005-01-29T00:00:00Z"),
                            "name" : "Japanese",
                            "school" : "Languages",
                            "score" : 98
                    },
                    {
                            "start_date" : ISODate("2005-01-29T00:00:00Z"),
                            "name" : "Information and Communication Technology (ICT)",
                            "school" : "Technology",
                            "score" : 76
                    }
            ]
    }
    
### RoboMongo

You may use whatever GUI you want, or none at all if you love the Mongo shell but I'll be using RoboMongo, which is [available for free](https://robomongo.org/) and is provided with a [GNU LGPL](https://www.gnu.org/licenses/lgpl.html) so you can use it with a clean conscience.

After installing it you'll be presented with an empty window for you to define a connection.

![](https://bytebucket.org/BanksySan/banksysan.content/raw/a5859dcc91f976de72cb9375f3f2d63eefefc804/mongodb-workshop/robomongo/robomongo-connections-window-empty.PNG)

Click an `Create` and you'll be presented with the `Connection Settings` dialogue.

![](https://bytebucket.org/BanksySan/banksysan.content/raw/a5859dcc91f976de72cb9375f3f2d63eefefc804/mongodb-workshop/robomongo/new-connection-window-connection-tab.PNG)

The default settings are all correct (unless you have changed something), I'll rename the connection to `default` though.

Click `Test` and you'll hopefully get a response telling you that everything's as expected.

![](https://bytebucket.org/BanksySan/banksysan.content/raw/a5859dcc91f976de72cb9375f3f2d63eefefc804/mongodb-workshop/robomongo/test-connection-passed.PNG)

Click `Close` on this window and then `Save` on the `Connection Settings` window and you should see the connection listed.

![](https://bytebucket.org/BanksySan/banksysan.content/raw/a5859dcc91f976de72cb9375f3f2d63eefefc804/mongodb-workshop/robomongo/robomongo-connections-window-default-connection.PNG)

Click `Connect` and you should be presented with a window on the left showing the databases and, if you drill into them, the collection we've created.

![](https://bytebucket.org/BanksySan/banksysan.content/raw/a5859dcc91f976de72cb9375f3f2d63eefefc804/mongodb-workshop/robomongo/tree-view-of-server.PNG)

Right clicking on the collection and selecting `View Documents` will give you the documents (paginated fifty per page) in one of three formats:

Tree view

![](https://bytebucket.org/BanksySan/banksysan.content/raw/a5859dcc91f976de72cb9375f3f2d63eefefc804/mongodb-workshop/robomongo/tree-view-of-documents.PNG)

Table view

![](https://bytebucket.org/BanksySan/banksysan.content/raw/a5859dcc91f976de72cb9375f3f2d63eefefc804/mongodb-workshop/robomongo/table-view-of-server.PNG)

Text view

![](https://bytebucket.org/BanksySan/banksysan.content/raw/a5859dcc91f976de72cb9375f3f2d63eefefc804/mongodb-workshop/robomongo/text-view-of-server.PNG)

You can swap between them with the function keys:

* F2 Tree view
* F3 Table view
* F4 Text view