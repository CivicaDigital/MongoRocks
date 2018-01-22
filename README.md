# _Mongo Rocks_

# Getting started
## Prerequisites
You'll need to have the following ready to go:

1. MongoDB (3.6 or above) -  Get is from the [MongoDB Download Centre](https://www.mongodb.com/download-center?community#community).
2. A GUI - There are a few, my favourite is [RoboMongo](https://robomongo.org/download).
3. The sample data from my [Git repo](https://bitbucket.org/BanksySan/mongorocks/downloads/).

## Running MongoDB
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

`mongo.exe` provides us with a shell to work with Mongo (there are drivers for languages and GUIs available as well, we'll work with the shell for a bit though).  By default the Mongo Shell will:

* Try to connect at `localhost:27027`.
* Will not authenticate.

So, we want the defaults so just run `mongo.exe`  Firstly see more logs appear on the server's output:

    [listener] connection accepted from 127.0.0.1:59218 #1 (1 connection now open)
    [conn1] received client metadata from 127.0.0.1:59218 conn: 
    { 
        application: { 
            name: 'MongoDB Shell' 
        }, 
        driver: { 
            name: 'MongoDB Internal Client', 
            version: '3.6.0' 
        }, os: { 
            type: 'Windows', 
            name: 'Microsoft Windows 10', 
            architecture: 'x86_64', 
            version: '10.0 (build 14393)' 
        } 
    }
    
> NB:  I've formatted the output so that it's readable.

We'll also see that the client has written some warning about this not being a recommended configuration as well as the command prompt change to indicate that its waiting for input.

# CRUD
## Databases & Collections

Data in Mongo is contained _collections_ which are in _databases_.  There can be many databases on any server and each database can contain many collections.

Firstly let's see what databases exist by default.

    > show dbs
    admin   0.000GB
    config  0.000GB
    local   0.000GB

These all have roles to play but are not used to store your documents.  In fact if you were to put your documents in any of these databases than they may well get deleted or the server might just break.

Creating databases and collections is done implicitly.  If you try to write to a database and\or collection that doesn't exist then it'll just get created.

When you're working in the shell you'll always to working in the context of a single database.  The reference to that database is stored in the `db` variable.  Just type `db` to see what it's pointing to:

    > db
    test

When the Mongo client first connects to a server it will always be in the context of the `test` database.  `test` doesn't exist, but that doesn't matter because it will be created when it's needed.

To change the database context you're working in (i.e. the value of `db`) use the `use <database_name>` command.

    > use mongo_rocks
    switched to db mongo_rocks

We can see what collections are in a database with `show collections`.  There aren't any yet, there will be soon.

## BSON Documents

BSON is the format that all Mongo documents are stored and retrieved data in.  Based on JSON, BSON is is nested structure that contains key/value pairs of objects, arrays and data types.  Unlike JSON, BSON is not a text file.  

BSON also supports more data types than JSON, these are described in the [MongoDB documentation](https://docs.mongodb.com/manual/reference/bson-types/) and in the [BSON Specification](http://bsonspec.org/spec.html).  In BSON language, the data structure is referred to as a _document_, as are all objects in the document (_nested documents_).

> JSON is not completely a subset of BSON as BSON doesn't allow `0x00` character.  BSON reserves `0x00` for control purposes such as string termination.


Take this BSON document:

    
    {
        '_id' : ObjectId('0123456789abcdef01234567'),
        'name' : 'David'
    }

`ObjectId` is not valid JSON.  This is a BSON object similar to a GUID, but optimised for generation speed over guaranteed uniqueness.  If a document is created without an `_id` field then Mongo will create one and assign it a new `ObjectId`.  

The BSON file created for this would have this structure:

| Description       | Bytes                                                       | Value                      |
|:------------------|:------------------------------------------------------------|:---------------------------|
| File size         | `26` `00` `00` `00`                                         | `38`                       |
| Data type         | `07`                                                        | _ObjectId_                 |
| Field name        | `5F` `69` `64`                                              | `_id`                      |
| String terminator | `00`                                                        |                            |
| Field Value       | `01` `23` `45` `67` `89` `AB` `CD` `EF` `01` `23` `45` `67` | `0123456789ABCDEF01234567` |
| Data type         | `02`                                                        | _String_                   |
| Field name        | `6E` `61` `6D` `65`                                         | `name`                     |
| String terminator | `00`                                                        |                            |
| Field Length      | `06` `00` `00` `00`                                         | `6`                        |
| Field value       | `44` `61` `76` `69` `64`                                    | `David`                    |
| String terminator | `00`                                                        |                            |
| Object terminator | `00`                                                        |                            |

> NB: There is a 64Mb limit on any document size.


Mongo provides tooling to export the document to JSON.  Mongo will produce valid JSON with a naming convention to capture data types that aren't in the JSON specification.

The above document would be serialised to JSON like so:

    {
        '_id': {
            '$oid':'0123456789abcdef01234567'
        },
        'name':'David'
    }

## Inserting Documents

Let's insert a document into a collection called `names` in a database called `mongo_rocks`:

    > use mongo_rocks
    switched to db mongo_rocks
    > db.names.insert({
    ... _id: NumberInt(1),
    ... name: 'David'
    ... })
    WriteResult({ 'nInserted' : 1 })

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
            'acknowledged' : true,
            'insertedIds' : [
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
            'nInserted' : 0,
            'writeError' : {
                    'code' : 11000,
                    'errmsg' : 'E11000 duplicate key error collection: test.names index: _id_ dup key: { : 2 }'
            }
    })

## Retrieving Documents
We use the find method to retrieve documents, or rather we use the cursor returned by the find function.  The function has the syntax:

    find(<filter>, <projection>)

Both the filter and the projection are optional.

To retrieve all the documents we just call the find function with either no arguments, or with an empty object.

    > db.names.find()
    { '_id' : 1, 'name' : 'David' }
    { '_id' : 2, 'name' : 'Alice' }
    { '_id' : 3, 'name' : 'Bob' }
    { '_id' : 4, 'name' : 'Charlie' }

The reason that we're seeing the documents, rather than seeing a cursor is because the client is figuring out that we probably want to get the documents, therefore it's iterating the documents with the cursor and displaying them without you having to tell it too.  We'll see later that this doesn't happen when there are a lot of documents returned.

## Projection
The projection object tells the server what fields we want to see.  We have the option to either give it a white-list or a black-list (not both).  For example, if we only wanted to see the `_id` field then we'd pass:

    > db.names.find({}, {
    ... _id: 1
    ... })
    { '_id' : 1 }
    { '_id' : 2 }
    { '_id' : 3 }
    { '_id' : 4 }

If we wanted to exclude the `_id` field then we'd pass a `0` instead:

    > db.names.find({}, {
    ... _id: 0
    ... })
    { 'name' : 'David' }
    { 'name' : 'Alice' }
    { 'name' : 'Bob' }
    { 'name' : 'Charlie' }

There is a _gotcha_ here,  The `_id` field is always returned unless you explicitly exclude it.  So to get only the names we need to pass both the `name` field and the `_id` field to the projection:

    > db.names.find({}, {
    ... _id: 0,
    ... name: 1
    ... })
    { 'name' : 'David' }
    { 'name' : 'Alice' }
    { 'name' : 'Bob' }
    { 'name' : 'Charlie' }
    
>  NB:  This is the only example of `1` and `0` appearing in the projection.

We can search for specific documents by providing a filter document:

    > db.names.find({
    ... name: 'David'
    ... }
    ... )
    { '_id' : 1, 'name' : 'David' }

## Removing Documents

Removing documents is similar to finding them.  We pass in a the same filter object as we would when finding and also an options object.  There is a is safety feature though, the filter object isn't optional (to prevent you from accidentally pressing of go and deleting the entire database).

    db.names.remove(<filter>)
    
## Updating Documents

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
    WriteResult({ 'nMatched' : 1, 'nUpserted' : 0, 'nModified' : 1 })
    
Notice the use of the `$set` operator field.  This tells Mongo that you want to insert a new field, without it Mongo would think that you wanted to replace the whole document.

    > db.names.find({
    ... name: 'Alice'
    ... })
    { '_id' : 2, 'name' : 'Alice', 'date_of_birth' : ISODate('1995-01-30T00:00:00Z') }
    

## Scripting

The Mongo client will execute arbitrary JavaScript.  For example, we could write a loop to add 1000 documents:
    
    > for(var i = 0; i < 1000; i++) {
    ... db.names.insert({number: NumberInt(i)});
    ... }
    WriteResult({ 'nInserted' : 1 })
    
When we search now we see lots of documents:

    > db.names.find({}, {_id: 0})
    { 'number' : 0 }
    { 'number' : 1 }
    { 'number' : 2 }
    { 'number' : 3 }
    { 'number' : 4 }
    { 'number' : 5 }
    { 'number' : 6 }
    { 'number' : 7 }
    { 'number' : 8 }
    { 'number' : 9 }
    { 'number' : 10 }
    { 'number' : 11 }
    { 'number' : 12 }
    { 'number' : 13 }
    { 'number' : 14 }
    { 'number' : 15 }
    { 'number' : 16 }
    { 'number' : 17 }
    { 'number' : 18 }
    { 'number' : 19 }
    Type 'it' for more

Now we have a good number of documents we can see the batching behaviour of the cursor, only twenty documents are returned.  We can iterate the cursor with the `it` command.
    
    > it
    { 'number' : 20 }
    { 'number' : 21 }
    { 'number' : 22 }
    { 'number' : 23 }
    { 'number' : 24 }
    { 'number' : 25 }
    { 'number' : 26 }
    { 'number' : 27 }
    { 'number' : 28 }
    { 'number' : 29 }
    { 'number' : 30 }
    { 'number' : 31 }
    { 'number' : 32 }
    { 'number' : 33 }
    { 'number' : 34 }
    { 'number' : 35 }
    { 'number' : 36 }
    { 'number' : 37 }
    { 'number' : 38 }
    { 'number' : 39 }
    Type 'it' for more

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
    
JavaScript is powerful and might be an unacceptable attack vector.  You can prevent JavaScript from being executed on the server by passing the `--noscripting` option when you start `mongod.exe`.  Scripting on the client is never prevented.

## Importing and Exporting

Mongo has two exporting tools and two importing tools, to be used in pairs:

* `mongoexport.exe` & `mongoimport.exe`
* `mongodump.exe` & `mongorestore.exe`

The first two work with JSON, the second two work with BSON and meta-data.  For a full backup/restore you need to use the second pair.  Not only will it include the data in native BSON but it will also include the indexes etc.

    ...>mongoexport --collection names --out .\names-export.json
    2018-01-16T15:49:21.203+0000    connected to: localhost
    2018-01-16T15:49:21.254+0000    exported 1000 records

This gives you a file like: 

    {'_id':{'$oid':'5a5e1554c469f5667ecde31c'},'number':0}
    {'_id':{'$oid':'5a5e1554c469f5667ecde31d'},'number':1}
    {'_id':{'$oid':'5a5e1554c469f5667ecde31e'},'number':2}
    ...
    {'_id':{'$oid':'5a5e1554c469f5667ecde703'},'number':998}
    {'_id':{'$oid':'5a5e1554c469f5667ecde703'},'number':999}
    
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

The `names.bson` file contains all out data in BSON format.  The `names.metadata.json` file contains information about the collection so that it can be rebuilt accurately, this contains:

    {
        'options': {},
        'indexes': [{
            'v': 2,
            'key': {
                '_id': 1
            },
            'name': '_id_',
            'ns': 'mongo_rocks.names'
        }],
        'uuid': 'dc47a6aa6a0c4c368688c1cc7fa5775f'
    }
    
We can see that there is a single index in this database called `_id_` and it's on the `_id` field ascending (descending would be a `-1` instead).

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

Now restore from the `mongo-rocks-dump` directory (download it compressed from [the repository](https://bitbucket.org/BanksySan/mongorocks/downloads/mongo-rocks-dump.v1.zip) and uncompress it). 


    ...> mongorestore --nsInclude mongo_rocks.students mongo_rocks\mongo-rocks-dump
    preparing collections to restore from
    reading metadata for mongo_rocks.students from mongo_rocks\mongo-rocks-dump\mongo_rocks\students.metadata.json
    restoring mongo_rocks.students from mongo_rocks\mongo-rocks-dump\mongo_rocks\students.bson
    no indexes to restore
    finished restoring mongo_rocks.students (10000 documents)
    done
    
Now open the Mongo shell again.  Have a look around and satisfy yourself that it's in good order:

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
            '_id' : 1,
            'personal' : {
                    'gender' : 'Male',
                    'given_names' : [
                            'Trent',
                            'Paul'
                    ],
                    'surname' : 'James',
                    'date_of_birth' : ISODate('1984-04-28T23:00:00Z')
            },
            'enrolments' : [
                    {
                            'start_date' : ISODate('2003-01-29T00:00:00Z'),
                            'name' : 'Psychology',
                            'school' : 'Humanities',
                            'score' : 27
                    },
                    {
                            'start_date' : ISODate('2003-01-29T00:00:00Z'),
                            'name' : 'Malay',
                            'school' : 'Languages',
                            'score' : 46
                    },
                    {
                            'start_date' : ISODate('2003-01-29T00:00:00Z'),
                            'name' : 'Product Design',
                            'school' : 'Technology',
                            'score' : 46
                    },
                    {
                            'start_date' : ISODate('2003-01-29T00:00:00Z'),
                            'name' : 'Social Science',
                            'school' : 'Humanities'
                    },
                    {
                            'start_date' : ISODate('2003-01-29T00:00:00Z'),
                            'name' : 'Law',
                            'school' : 'Humanities',
                            'score' : 67
                    },
                    {
                            'start_date' : ISODate('2004-01-29T00:00:00Z'),
                            'name' : 'History',
                            'school' : 'Humanities',
                            'score' : 84
                    },
                    {
                            'start_date' : ISODate('2004-01-29T00:00:00Z'),
                            'name' : 'Home Economics: Food and Nutrition',
                            'school' : 'Humanities'
                    },
                    {
                            'start_date' : ISODate('2004-01-29T00:00:00Z'),
                            'name' : 'Classical Greek',
                            'school' : 'Languages',
                            'score' : 93
                    },
                    {
                            'start_date' : ISODate('2004-01-29T00:00:00Z'),
                            'name' : 'Electronics with Resistant Materials',
                            'school' : 'Technology',
                            'score' : 54
                    },
                    {
                            'start_date' : ISODate('2005-01-29T00:00:00Z'),
                            'name' : 'Gujarati',
                            'school' : 'Languages',
                            'score' : 45
                    },
                    {
                            'start_date' : ISODate('2005-01-29T00:00:00Z'),
                            'name' : 'Japanese',
                            'school' : 'Languages',
                            'score' : 98
                    },
                    {
                            'start_date' : ISODate('2005-01-29T00:00:00Z'),
                            'name' : 'Information and Communication Technology (ICT)',
                            'school' : 'Technology',
                            'score' : 76
                    }
            ]
    }
    
## RoboMongo

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

* `F2` Tree view
* `F3` Table view
* `F4` Text view

# Managing Documents

## Sample Data
We're working with a pre-built collection called `students`.  There are `1000` documents containing personal information and information about courses they've enrolled in.

The basic structure looks like this, though this can change from document to document:

    {
        '_id' : NumberInt,
        'personal' : {
            'gender' : String,
            'given_names' : [ 
                String
            ],
            'surname' : String,
            'date_of_birth' : ISODate,
        },
        'enrolments' : [{
            'start_date' :ISODate,
            'name' : String,
            'school' : String,
            'score' : NumberInt
        }]
    }

We're going to be going through working with these documents in some detail, with exercises thrown in.

## Find by Value

As previously mentioned, every document in Mongo must have a unique primary key stored in the `_id` field.  To keep things easy I've given all the students an integer key from `1` to `1000`.

### Exercise One:  Find by ID

1. Find the document with an `_id` of 50. (You should return Trudy Alice James)

We can also use regular expressions (though these will be unsargable).

## Nested Documents

Nested documents present our first challenge.  Let's get all the female students, intuition may suggest that you can write a query like this:

    db.students.find({
        personal: {
            gender: 'Female'
        }
    })

However, if you run this then you'll only return one document that I've specifically put in so that it matches this query.

    {
        _id : 100,
        personal : {
            gender : 'Female'
        },
        enrolments: [ ... ]
    }
    
The reason that this document is returned is because this structure tells Mongo that you want an exact match, that being a nested document in the `personal` field that contains only the `gender` field with the value `Female`.  The query we want is to find all the documents which have a nested `personal` document that contains a `gender` field with the value `Female`.  To do this we use the dot syntax:

    db.students.find({
        'personal.gender': 'Female'
    })

### Exercise Two:  Querying Nested Documents and Regular Expressions

1. Search for all users who's surname starts with `A`.  (There's 607)
1. Find all the students with the surname of `Roberts`.

## Arrays

Our student documents have two arrays, on contains strings, the other more documents.  Because we can't ever guarantee the data type of a field (or even it's existence), Mongo has some special rules for dealing with arrays of things.

Consider the following three documents (`_id` removed):

    /* 1 */
    {
        "foo" : 1
    }
    
    /* 2 */
    {
        "foo" : [ 
            1
        ]
    }
    
    /* 3 */
    {
        "foo" : [ 
            3, 
            2, 
            1
        ]
    }
    
All of them have the integer `1` in the `foo` field, in some it's inside an array.  When we use the filter `{ foo: 1 }` however we actually return all three, this is because when Mongo reaches an array it will iterate through it looking for any match in the array.

If we want to find an element in a specific position in an array then we can use an ordinal position with a dot delimiter `{ 'foo.0': 1 }`.  This will only return the documents where the the `1` is first element in the array.

### Exercise Three:  Select array elements

1. Find all the students with `Judy` as any of their names. (There's 582.)
1. Find all the students with `Judy` as their first name. (There's 266.)
1. Find all the students with `Judy` as their first name and `Grace` as their second name. (There's 17.)

Arrays can also contain documents, which can be filtered on.  If we want to get everyone who's got a score of `50` then we can use `{ 'enrolments.score': 50 }`.  If we want to find everyone who's got a score of `50` in the `Systems and Control Technology` course then we have a problem.

    {
        'enrolments.score': 50,
        'enrolments.name': 'Systems and Control Technology'
    }
    
This query object will search for all documents that have a score of `50` in the enrolments array and have a name of `Systems and Control Technology` in the enrolments array.  We want both these values to be in the same document.  To achieve this we need to use an operator, `$elemMatch`.  `$elemMatch` _wraps_ the fields that you want to be constrained to a single document.

    {
        enrolments: {
            $elemMatch: {
                score: 50,
                name: 'Systems and Control Technology'
            }
        }
    }

When we run this we now get the expected result.

### Exercise Four:  Select array documents

1. Find all the students called `Victor` who enrolled in any course in the `Humanities` school and got a score of `31` in the course.  (There's 10)


## Logical Operators

We need to be able to perform logical operations as well, so for we've only been matching on value which won't get us very far in the real world.  The logical (boolean) operators are documented in the [Logical Query Operations](https://docs.mongodb.com/manual/reference/operator/query-logical/) documentation.

| Name   | Description                                                                                             |
|--------|---------------------------------------------------------------------------------------------------------|
| `$and` | Joins query clauses with a logical AND returns all documents that match the conditions of both clauses. |
| `$not` | Inverts the effect of a query expression and returns documents that do not match the query expression.  |
| `$nor` | Joins query clauses with a logical NOR returns all documents that fail to match both clauses.           |
| `$or`  | Joins query clauses with a logical OR returns all documents that match the conditions of either clause. |

The not clause deserves some special mention, it is the set of documents not returned by an inclusive search (i.e. not, not the documents).  When it comes to arrays the behaviour can be a bit unintuitive.  If you recall, when searching for a value against an array field, Mongo will iterate all the values looking for any match.  The not operator looks through all the values in the array checking that all the elements satisfy the not clause.

Consider this document:

    {
        "foo" : [ 
            1, 
            2, 
            3
        ]
    }

If I search with an equality I get the document returned:

    db.test.find({
        foo: 2
    })

As we know, this is because Mongo finds the searched for value in the array.  If I now search with a not clause instead then I get no document returned:

    db.test.find({
        foo: {
            $not: {
                $eq: 2
            }
        }
    })

Certainly there are elements in the array that are not `2`.  The reasoning for this seemingly strange change in searching behaviour to satisfy that all the documents returned from a query, summed with all the documents that are returned from `$not` that query should equal the total number of documents.  If this behaviour wasn't true then we could have a query and not that query both returning all the documents.

## Exercise Five: Not clauses

1. Find all the students who haven't enrolled in any course from the `Technology` school. (There's 3592)
2. Find all the students who have scored a `0` in any subject from the `Technology` school and then all those who haven't.  Confirm that these both add up to the total number of documents. (There's 8548 who have not and 1452 who have.) 

## Comparison documents

Matching by equality isn't usually enough, we want often to match by comparison. Mongo provides us with a set of comparison  operators to allow us to perform comparisons.

|Name    | Description                                                          |
|--------|----------------------------------------------------------------------|
| `$eq`  | Matches values that are equal to a specified value.                  |
| `$gt`  | Matches values that are greater than a specified value.              |
| `$gte` | Matches values that are greater than or equal to a specified value.  |
| `$in`  | Matches any of the values specified in an array.                     |
| `$lt`  | Matches values that are less than a specified value.                 |
| `$lte` | Matches values that are less than or equal to a specified value.     |
| `$ne ` | Matches all values that are not equal to a specified value.          |
| `$nin` | Matches none of the values specified in an array.                    |

We could find all the students who were born after 1995 with the `$gte` operator.

    {
        'personal.date_of_birth' : {
            $gte: ISODate('1995-01-01')
        }
    }

The operators are described in Mongo's [Comparison Query Operators](https://docs.mongodb.com/manual/reference/operator/query-comparison/) documentation.

### Exercise Six: Comparison
1. Find all the students who were born after in or after 1995

### Array Operators

We have a few operators to help us interrogate arrays specifically.


| Name         | Description                                                                                         |
|--------------|-----------------------------------------------------------------------------------------------------|
| `$all`       | Matches arrays that contain all elements specified in the query.                                    |
| `$elemMatch` | Selects documents if element in the array field matches all the specified `$elemMatch`  conditions. |
| `$size`      | Selects documents if the array field is a specified size.                                           |

To find all the students who have both `Trudy` and `Alica` in their name we'd use:

    {
        'personal.given_names' : {
            $all: ['Trudy', 'Alice']
        }
    }

### Exercise Seven: Array Queries

1. Find all the students who are scored a `0` in both `Welsh` and `Dutch`.  (There's 4)
2. Find all the students who are scored a `0` in either `Welsh` and `Dutch`.  (There's 284)
1. Get the students who have no middle names.  (There's 967)

# Updating Documents
Updating documents is an interesting problem, both syntax wise and implementation wise.  How often you update documents will determine how you structure your database as well as determine what sort of data storage engine you choose (Mongo supports WiredTiger and MMAPv1).  We're going to stick to syntax here though and leave data storage implementation for some other time.

The update method accepts three arguments (the last is optional):

    db.collection.update(
       <query>,
       <update>,
       {
         upsert: <boolean>,
         multi: <boolean>,
         writeConcern: <document>,
         collation: <document>,
         arrayFilters: [ <filterdocument1>, ... ]
       }
    )

The query determines which documents should be be updated, the update document determines how they are to be updated.

> NB:  By default the update method will only update the first document it finds, to update all the documents we need to pass `{ multi: true }` in the final argument.

Consider this document:

    {
        _id : 'a',
        foo : 1.0,
        bar : 2.0
    }

Let's update `foo` to `2.0`.  Constructing the query is exactly the same as when we're finding documents.

    update({  _id: 'a' }, 
           {  foo: 2.0 })

When we check the document to see the changes though we see that something has gone wrong.

    {
        "_id" : "a",
        "foo" : 2.0
    }

The reason is that passing in a _vanilla_ document to the update query tells Mongo that you want to completely replace the document with the new one, only keeping the `_id`.

In fact, you can't change the `_id` field at all.

    update({
        _id: 'a'
    }, {
        _id: 'b'
    })

Will return error message:

    After applying the update, the (immutable) field '_id' was found to have been altered to _id: "b"

Normally, when updating documents we'll be wanting to manipulate parts of the document, adding, removing and changing fields.

The documentation for the operators is on the [Update Operators](https://docs.mongodb.com/manual/reference/operator/update/) page of the documentation. 

| Name              |                                                                                                                                               |
|-------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| `$currentDate`    | Sets the value of a field to current date, either as a Date or a Timestamp.                                                                   |
| `$inc`            | Increments the value of the field by the specified amount.                                                                                    |
| `$min`            | Only updates the field if the specified value is less than the existing field value.                                                          |
| `$max`            | Only updates the field if the specified value is greater than the existing field value.                                                       |
| `$mul`            | Multiplies the value of the field by the specified amount.                                                                                    |
| `$rename`         | Renames a field.                                                                                                                              |
| `$set`            | Sets the value of a field in a document.                                                                                                      |
| `$setOnInsert`    | Sets the value of a field if an update results in an insert of a document. Has no effect on update operations that modify existing documents. |
| `$unset`          | Removes the specified field from a document.                                                                                                  |
| `$addToSet`       | Adds elements to an array only if they do not already exist in the set.                                                                       |
| `$pop`            | Removes the first or last item of an array.                                                                                                   |
| `$pull`           | Removes all array elements that match a specified query.                                                                                      |
| `$push`           | Adds an item to an array.                                                                                                                     |
| `$pullAll`        | Removes all matching values from an array.                                                                                                    |




## Setting and Un-setting

The two primary operators for updating are the `$set` and `$unset` operators which will set a fields value (creating it if necessary) and removed a field, respectively.

In our previous example the update document we should have used was:

    update({
        _id: 'a'
    }, {
        $set: {
            foo: 2.0
        }
    })

This would have altered the document instead of replacing it.  Creating new fields is exactly the same:

    update({
        _id: 'a'
    }, {
        $set: {
            foo: 2.0,
            baz: 3.0
        }
    })
    
The document now has a new field:

    {
        "_id" : "a",
        "foo" : 2.0,
        "bar" : 2.0,
        "baz" : 3.0
    }

We can combine removing and setting fields into a single update:

    update({
        _id: 'a'
    }, {
        $set: {
            foo: 'foo',
            foz: 'foz'
        }, 
        $unset: {
            bar: 'it doesn\'t matter',
            baz: 'what value goes in an unset.'
        }
    })
    
> NB:  Any value can go after fields in the unset operator.
            
 The document not looks like:

    {
        "_id" : "a",
        "foo" : "foo",
        "foz" : "foz"
    }

### Exercise Eight: Remove Empty Enrolments

1. Many students are on the system who haven't actually enrolled in anything.  Delete the enrolment field from these students and add a new field to all students that don't have any enrolments to tell us that these students shouldn't be included from alumni communications.  (There's 2596)

(Hint, use `find` to check your query before updating)

## Updating Arrays

Update can also add and remove values from arrays, consider these documents:

    {
        _id: 1,
        foo: [1,2,3,4,5],
        bar: [3,4,5,6,7]
    }, {
        _id: 2,
        foo: [3,4,5,6,7],
        bar: [5,6,7,8,9]
    }

We can use the `$push` and `$pull` to add and remove elements:

    db.collection.update({
    }, {
        $pull: {
            foo: 3
        },
        $push: {
            bar: 'a'
        }
    },
    { 
        multi: true 
    })
    
Will now give us:
    
    /* 1 */
    {
        "_id" : 1.0,
        "foo" : [1.0, 2.0, 4.0, 5.0 ],
        "bar" : [3.0, 4.0, 5.0, 6.0, 7.0, "a"] }
    
    /* 2 */
    {
        "_id" : 2.0,
        "foo" : [4.0, 5.0, 6.0, 7.0 ], 
        "bar" : [6.0, 7.0, 8.0, 9.0, "a"] 
    }
    
> NB: You cannot push and pull from the same array in the same operation.

### Exercise  Nine: Why the floats?
1. If you haven't figured it out yet, find out why the integers are being converted to floating points when we perform these calculations on them.

We can also perform limited arithmetic calculations, increment and  multiplication.

Consider this document:

    {
        _id : 1,
        foo : 1,
        bar : 2
    }

I can increment the values in `foo` and `bar`, both positively and negatively:

    {
        $inc: {
            foo: 99,
            bar: -102
        }
    }

This will update the document to:

    {
        "_id" : 1,
        "foo" : 100.0,
        "bar" : -100.0
    }

> NB:  Notice that the type has changed to a float.


The `$max` and `$min` operators will replace a value only if it is more or less than the one given.  Given these two document:

    /* 1 */
    {
        "_id" : 1,
        "foo" : 1
    }
    
    /* 2 */
    {
        "_id" : 2,
        "foo" : 2
    }

We can use the `$min` operator to change the value if it is less than `-1`:

    {
        $min: {
            foo: -1
        }
    }
    
    
This will replace the `foo` in the first document, but not in second:

    /* 1 */
    {
        "_id" : 1,
        "foo" : -1.0
    }
    
    /* 2 */
    {
        "_id" : 2,
        "foo" : 2
    }
    

## Array Positions

Mongo has three positional operators (two prior to version 3.6).  These give you some control over what elements in an array are updated.

| Name              | Description                            |
|-------------------|---------------------------|
| `$`               | Acts as a placeholder to update the first element that matches the query condition.                                                           |
| `$[]`             | Acts as a placeholder to update all elements in an array for the documents that match the query condition.                                    |
| `$[<identifier>]` | Acts as a placeholder to update all elements that match the arrayFilters condition for the documents that match the query condition.          |

The first is the positional operator `$`.  This will match the first element that is matched in the search.

Consider this document:

    {
        "foo" : [ 
            1, 
            2, 
            2, 
            3
        ]
    }


If we craft a filter to search for a specific element in the array then the `$` will reference the first located element:

    update({
        foo: 2
    }, {
        $inc: {
            'foo.$': 10
        }
    })

The element

    {
        "foo" : [ 
            1, 
            12.0, 
            2, 
            3
        ]
    }

The first `2` is incremented, the subsequent matches are not incremented.

### Exercise Ten: Matching with $
1. Remove the score from the first instance of a course from the School of Languages.  (There are 7326)

We can also update all the documents in an array if one matches. 

    /* 1 */
    {
        "_id" : ObjectId("5a6082569d09cf98319ae41c"),
        "foo" : [ 
            1, 
            2, 
            3, 
            4
        ]
    }
    
    /* 2 */
    {
        "_id" : ObjectId("5a608a54cf1ebfa986959538"),
        "foo" : [ 
            1, 
            3, 
            4, 
            5
        ]
    }

Use this update:

    update({foo: 2}, {
        $mul: {
            'foo.$[]': 10,
        }
    })

We have multiplied all the elements in `foo` in document one, none in document two.


    /* 1 */
    {
        "_id" : ObjectId("5a6082569d09cf98319ae41c"),
        "foo" : [ 
            10.0, 
            20.0, 
            30.0, 
            40.0
        ]
    }
    
    /* 2 */
    {
        "_id" : ObjectId("5a608a54cf1ebfa986959538"),
        "foo" : [ 
            1, 
            3, 
            4, 
            5
        ]
    }

The last, and most recently introduced positional update operator is the array filter.  It's actually a two part operation, we declare an identifier for an array in out update, we can then supply a further filter in the options object.

Conciser this document:

    {
        "results" : [ 
            {
                "name" : "foo"
            }, 
            {
                "name" : "bar"
            },
            {
                "name": "baz"
            }
        ]
    }
    
We can write a query that will match `foo` and `baz`:

**I'm still figuring this one out.  Watch this space**

# Aggregation

Mongo DB has three mechanisms for aggregation.

1. Aggregating with the cursor returned by a find.
2. The Aggregation Pipeline.
3. Map-Reduce.

## Aggregating with the find method

As mentioned earlier, the find method doesn't return the records directly; instead it returns a cursor which will return the records in batches as we iterate over it.  The cursor also exposes some methods that allow us to modify, aggregate and/or replace the documents.  The full list is in the [documentation](https://docs.mongodb.com/manual/reference/method/js-cursor/), there are more that just aggregation methods.

The aggregation methods are:

| Method              | Description                                                                                                   |
|---------------------|---------------------------------------------------------------------------------------------------------------|
| `cursor.count()`    | Modifies the cursor to return the number of documents in the result set rather than the documents themselves. |
| `cursor.limit()`    | Constrains the size of a cursor’s result set.                                                                 |
| `cursor.map()`      | Applies a function to each document in a cursor and collects the return values in an array.                   |
| `cursor.size()`     | Returns a count of the documents in the cursor after applying skip() and limit() methods.                     |
| `cursor.skip()`     | Returns a cursor that begins returning results only after passing or skipping a number of documents.          |
| `cursor.sort()`     | Returns results ordered according to a sort specification.                                                    |
| `cursor.toArray()`  | Returns an array that contains all documents returned by the cursor.                                          |

### Exercise Eleven: Sorting
1. (This is a trick question, but what's the trick?)  Return the 10 oldest students, in alphabetical surname order, by the number of given names they have in reverse order.
2. Use map to write out each student's full name.

## The Aggregation Pipeline

Finally, we're at the good stuff.  The Aggregation Pipeline allows you to transform documents as they flow through a number of stages.  The output of each stage becomes the input to the next.  Each stage both accepts and produces one of more BSON documents.

The Aggregation Pipeline is implemented in the aggregate method:

    aggregate([{...}, {...}, ...])

Each object is a BSON document that describes one stage.  A list is on the Mongo [documentation](https://docs.mongodb.com/manual/meta/aggregation-quick-reference/).

The simplest stage is probably the `$count` stage.  It simple accepts a field name to output the count of all the documents it see into and produces a single document with this field.

    aggregate([{
        $count: 'my_count'
    }])
    
This will produce

    {
        my_count: 12345
    }

By itself it will give us the total number of documents in the collection, to get finer control over what we count we can first filter the documents coming through with the `$match` stage.

    aggregate([{
        $match: {
            <condidtion>
        }
    }, {
        $count: 'my_count'
    }])

This will product the same structured document as the last example, but only the documents that get through the `$match` stage will be counted.

### Exercise Twelve: Counting
1. Count how many men there are.
2. Count how many women there are.
3. Is anyone neither male nor female?

We can also sort with the `$sort` stage.

### Exercise Thirteen: Sort and match
1. Return the top ten documents, sorted by the number of courses they have enrolled in (ignoring whether they completed the course or not).
1. Return the next ten documents.

The first powerful stage is the `$projection` stage.  This lets us change the documents as they come through, adding, removing and changing fields.  it's basic operation is the same as the projection object in the find method, that being you can either have a white list, or a black list, signified with `1` or `0` (`_id` still need to be explicitly excluded.

### Exercise Fourteen: Limiting with Projection
1. Output only the personal details for ten students.

We can also manipulate fields and reference other fields.

Consider these documents:

    /* 1 */
    {
        "_id" : 1,
        "foo" : "a"
    }
    
    /* 2 */
    {
        "_id" : 2,
        "foo" : "b"
    }
    
    /* 3 */
    {
        "_id" : 3,
        "foo" : "c"
    }
    
We can use projection to produce a new document that creates a new document using the fields in the input documents.  To refer to the value of another field we use a dollar syntax.  To refer to the value of `foo`, for example, we'd write `$foo`.

So we can write the following pipeline:

    db.collection.aggregate([{
        $project: {
            _id: 0,
            foo: 1,
            new_field: '$_id',
            another_new_field: '$foo',
            message: 'Hello World!'
        }
    }])
    
This will give us a new stream of documents:

    /* 1 */
    {
        "foo" : "a",
        "new_field" : 1,
        "another_new_field" : "a",
        "message" : "Hello World!"
    }
    
    /* 2 */
    {
        "foo" : "b",
        "new_field" : 2,
        "another_new_field" : "b",
        "message" : "Hello World!"
    }
    
    /* 3 */
    {
        "foo" : "c",
        "new_field" : 3,
        "another_new_field" : "c",
        "message" : "Hello World!"
    }

This is that's happening:

| Expression                | Effect                                                     |
|---------------------------|------------------------------------------------------------|
| `_id: 0`                  | Exclude this field                                         |
| `foo: 1`                  | Include this field                                         |
| `new_field: '$_id'` <br /> `another_new_field: '$foo'` | Create new fields and assign them the values in `_id` and `foo` respectively |
| `message: 'Hello World!'` | Create a new field and give it the value of `Hello World!`

### Exercise Fifteen:  More projection
1. Return a document with the student's first name and surname in a field called `name` and another field called `enrolment_count` which contains the number of courses they are enrolled in.

## Grouping

The `$group` stage performs aggregations by grouping and producing a document for each group.  The group is determined by how you define the `_id` field.  Each unique `_id` will be a new group.

I have a sequence of document with the following values:

    /* 1 */
    {
        "foo" : "a"
    }
    
    /* 2 */
    {
        "foo" : "b"
    }
    
    /* 3 */
    {
        "foo" : "c"
    }

These have been duplicated in the collection.  I can use `$group` to find out information about them.

I can perform a simple distinct operation by just setting the `_id` field to the value of the field we want to get distinct values of:

    db.collection.aggregate([{
        $group: {
            _id: '$foo',
        }
    }])
    
Gives us a document for `a`, `b` and `c`:

    /* 1 */
    {
        "_id" : "b"
    }
    
    /* 2 */
    {
        "_id" : "c"
    }
    
    /* 3 */
    {
        "_id" : "a"
    }
    
> NB:  Notice that ordering isn't preserved.

We can do more using the various operators available, we can count the number of documents there by giving a value of `1` to the `$sum` operator:

    db.collection.aggregate([{
        $group: {
            _id: '$foo',
            count: {
                $sum: NumberInt(1)
            }
        }
    }])
    
This now gives us:

    /* 1 */
    {
        "_id" : "b",
        "count" : 55
    }
    
    /* 2 */
    {
        "_id" : "c",
        "count" : 55
    }
    
    /* 3 */
    {
        "_id" : "a",
        "count" : 55
    }

### Exercise Sixteen: Counting and Sorting Given Names
1. Find out the count of students with each number of given names, sort these.
1. How do the number of given names a student has relate to their surname?

We can also aggregate values into an array using the `$push` operator.

Consider this documents:
    
    {
        "foo" : "a",  // foo can be 'a' or 'b'.
        "bar" : 1.0  // bar is a number
    }

We can use `$group` to create documents for `a` and `b` that contain all the values of `bar` for those documents.

    db.collection.aggregate([{
        $group: {
            _id: '$foo',
            bars: {
                $push: '$bar'
            }
        }
    }])
    
This will give us:
    
    /* 1 */
    {
        "_id" : "b",
        "bars" : [ 
            10.0, 
            11.0, 
            12.0, 
            14.0, 
            15.0
        ]
    }
    
    /* 2 */
    {
        "_id" : "a",
        "bars" : [ 
            1.0, 
            2.0, 
            3.0, 
            4.0, 
            5.0
        ]
    }

### Exercise Seventeen: Creating a New Array

1. Create documents each having a surname and all the first names that people with that surname could have.
1. Do the same, but this time with no duplicate names.

## Unwinding (pivoting)

The `$unwind` stage allows you to break an array in a new document for each array entry.

Consider this document:

    {
        "_id" : 1,
        "foo" : "abc",
        "bar" : [ 
            1.0, 
            2.0, 
            3.0
        ]
    }

We have an array in the `bar` field.  By unwinding this array we will produce three documents, one for each `bar` value.

    db.collection.aggregate([{
        $unwind: '$bar'
    }])

This gives us the following:

    /* 1 */
    {
        "_id" : 1,
        "foo" : "abc",
        "bar" : 1.0
    }
    
    /* 2 */
    {
        "_id" : 1,
        "foo" : "abc",
        "bar" : 2.0
    }
    
    /* 3 */
    {
        "_id" : 1,
        "foo" : "abc",
        "bar" : 3.0
    }

> NB: Notice that the `_id` is copied as well, therefore we couldn't write this directly back to the database.

## Exercise Eighteen: Unwinding

1. What is the most popular course?
1. List all the courses by their school.
1. That are the total points awarded for each school?
1. Produce the same data again, but this time in a single document.

# Indexes
Like any database worth its salt, Mongo supports indexing.  Mongo has several types of indexes, some will be familiar if you've used any RDBMS and some are unique.

We can view all the indexed with either the `db.collection.getIndices()` or the `db.collection.getIndexes()`. command.  For our collection it'll just return one index:


    [
        {
            "v" : 2,
            "key" : {
                "_id" : 1
            },
            "name" : "_id_",
            "ns" : "test.collection"
        }
    ]


Because Mongo doesn't enforce a collection-wide schema (bar the existence of `_id` the fields we index on don't have to assume any.

The above shows if that the index called `_id_` is on a field called `_id` and is an ascending index, a descending index would have `-1`.  the `ns` field is the namespace of the collection.

The `v` field indicates the version of the implementation of the index, this is version 3.

> Version 1 (i.e. `v: 0`) indexes are no longer supported.

Mongo doesn't let you create duplicate indexes (i.e. identically configured indexes).

Like other databases, Mongo has a query plan cache which it will check before generating a new plan.

We can see details of the chosen strategy with the `explain` function.  It's available off either the cursor returned from a collection CRUD operation, or directly off the collection.

    db.students.find({
        _id: 50
    }).explain()
    
or
    
    db.students.explain().find({
            _id: 50
        })

The latter for is normally preferred as there are some explainable operations that don't return an object that has the explain function.

>NB RoboMongo doesn't support he latter form so, if this is needed, you either have to use the Mongo shell.

## The Primary Index
You've seen that every document has a primary in the `_id` field, the name of this index is `_id`'.  You cannot remove this index.  Indexes can't be edited in Mongo, instead you have to drop and create, this prevents editing the `_id_` index.  The primary index is also the clustered index and is implicitly a unique index, even though it doesn't say it's unique in its output.

The primary index is also a special case because it enforces that `_id` exists.

## Index Types
Mongo supports several types of indexes including simple values, unique indexes, capped indexes, and time-to-live indexes.


| Type         | Description                                                                                            |
|:-------------|:-------------------------------------------------------------------------------------------------------|
| Value        | Index for searching only.  No constraints.                                                             |
| Unique       | Ensures uniquness in the indexed values.  Null values can be excluded. (Arrays need special attention) |
| TTL          | Time to live, documents expire after a given time                                                      |
| 2D Sphere    | Allows searching on coordinates of the surface of a sphere.                                            |
| 2D           | Alows searching on a infinite, 2D membrane                                                             |
| Geo-Haystack | Allows searching on buckets on an infinate 2D membrane                                                 |
| Text         | Allows for culture aware text indexing                                                                 |

## Creating Indexes

Indexes are created with the `createIndex` function.  The function accepts two arguments, the first defined the fields the index should apply to, the second contains options relating to the index.  The second argument is optional.

The most basic create index function is:

    db.collection.createIndex({
        foo: 1
    })
    
This gives the following result:

    {
        "createdCollectionAutomatically" : false,
        "numIndexesBefore" : 1,
        "numIndexesAfter" : 2,
        "ok" : 1.0
    }

It indicates that the index was created successfully and that the total number of indexes has gone from 1 to 2 (the original index being `_id_`).  The details of the index are:

    {
        "v" : 2,
        "key" : {
            "foo" : 1.0
        },
        "name" : "foo_1",
        "ns" : "test.collection"
    }

You can see that a name for the index has been generated.  If we wanted to declare a name then we could have passed `{ name: 'my_index_name' }` as the second argument.

> NB, indexed can be read either forwards or backwards.

### Exercise Eighteen: Single Field Index (Scalar values)
1. Create an index on the surname field and perform a search.  Confirm that the expected index is used.
1. Search on the `date_of_birth` field.  What strategy is used?
1. Get all the documents ordered by `surname`, then by `date_of_birth`.  What strategy is used now? 
1. Get all the documents ordered by `date_of_birth`, then by `surname`.  What strategy is used now?

## Compound indexes

We can create compound indexes by listing the fields we want included along with the direction.

    db.collection.createIndex({
        foo: 1,
        bar: -1
    })
    
This will create a compound index.

    {
        "v" : 2,
        "key" : {
            "foo" : 1.0,
            "bar" : -1.0
        },
        "name" : "foo_1_bar_-1",
        "ns" : "test.collection"
    }

The index is ascending on `foo` and descending  on `bar`.

### Exercise Nineteen: Create a compound index.
1. Create a compound index over `gender` ascending and `date_of_birth` ascending.
1. Perform searches on these, when is the index used?
1. Sort on `gender` descending and `date_of_birth` descending, is the index used?
1. Sort with `gender` ascending and `date_of_birth` descending, is the index used?
1. Sort on `gender` only, are any indexes used?
1. Sort on `date_of_birth` only, are any indexes used?
1. Create a new index, this time with `gender` ascending and `date_of_birth` descending and repeat the above tests.

## Indexing Non-Scalar Values

Values in Mongo are often arrays, document or both.  Indexing these poses a new challenge for Mongo and for yourselves in constructing a good index.

Accounting for arrays is a particular problem.  When Mongo finds an array as a value it will index each field individually, this seems logical enough but it has some interesting side affects.

If a compound index contains arrays then there can only by one array per document.

Consider these documents:

    {
        "foo" : [1]
    }
    
    { 
        "bar" : [1] 
    }

I can add a compound index over these fields and nothing fails, however, if I try to add a document with both those fields, both with array values then I will get an error and the insert will fail:
    
    db.index_tests.insert({foo:[1], bar:[1]})
    WriteResult({
            "nInserted" : 0,
            "writeError" : {
                    "code" : 171,
                    "errmsg" : "cannot index parallel arrays [bar] [foo]"
            }
    })

The reasoning for this is complexity, the cross product of two arrays could quickly become too large to manage.

Also, arrays behave differently with unique indexes.  Assume we have a unique index on `foo`.  The following two documents would be a conflict

    {
        foo: 1
    }
    
    {
        foo:[1]
    }

However, the following would not cause a conflict:

    {
        foo: [1, 1]
    }

### Exercise Twenty
1. Why can't a unique index go used on the `personal.given_names`, given that each student never has multiple identical names?
1. Why are duplicate entries into an array field allowed, even when there is a unique index on the field?

## Time-To-Live Indexes

TTL indexes are quite different to other indexes.  These indexes can be used for optimising queries, and will be if they can, however their purpose is to purge documents.  We configure a TTL index with an `expireAfterSeconds` value and a field to monitor.  If a document has this field, and the date in it is older that the number of seconds configured then the document will get deleted.

> NB:  The document will get deleted when the worker process gets around to it.  You can't use this for time critical purging operations.

If the field doesn't contain a date, or doesn't exist then the document will never expire. 

>NB: If the document contains an array then the lowest date in the array (if there is one) will be used.

### Exercise Twenty-One: TTL Indexes
1. Create a TTL index, prove that the index will use the lowest date in an array.


# Capped Collections

Capped collections are similar to queues or the unix `tail` command.  It's a rolling set with a maximum number and total size of documents allowed, the oldest documents will be dropped to keep the number of documents at the capped amount.

Capped collections require a maximum number of bytes because it will reserve that much space, the maximum number of documents is an optional extra.

To create a capped collection we use the `createCollection` function and pass it an options object describing the collection.  For example, to create a collection capped called `capped` with 4096 bytes (the minimum number of bytes supported) we'd use this command:
    
    db.createCollection('capped', {
        capped: true,
        size: 4096
    })



### Exercise Twenty-Two: Prove that a Collection is Capped  
1. Write a script to prove that the collection is capped (use the `print(string)` function to write out to the console).
1. Create a collection capped by document count and then write a script to prove it.



