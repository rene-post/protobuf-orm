# Protobuf ORM

The protocol buffers object relational mapper (protobuf-orm) allows you to store
protobuf objects in a database.

This document starts with a short introduction to _protobuf_ followed by a short
description of what an _orm_ does. Subsequently there is a section containing an
example of how a protobuf object is stored in the database. After that there are
code snippets illustrating how the application code should interact with
protobuf-orm api to create and drop tables, to enumerate, read, update and
delete records.

## Protobuf

Protobuf (short for Protocol buffers) is a way of encoding structured data in
an efficient and extensible format. The data is described in so called .proto
files from which code is generated that can be used to store the data in
memory. The generated code also creates a reflection interface called a
description. The description allows application code to reason about the
structured data in memory.

An example of a protobuf message is given below.

```protobuf
message Foo {
	optional string text = 1;
	repeated int32 numbers = 2;
}
```

So these four lines declare a message named _Foo_ that contains two fields. The
first field named _text_ is a string. The second field named _numbers_ is a
collection of integers.

The main concept in protobuf is a structured data type named _message_. A
_message_ itself can contain fields that are either primitive value types or
other messages. Fields can be flagged as **required**, **optional** or
**repeated**. The repeated flag illustrates that the concept of a collection of
values is an inherent part of the protobuf specification language.

As shown in the example above, every field in a message has to be tagged with
a unique index (starting at 1). Additionally a field can have a _default_
value and additional _options_. Protobuf allows _options_ to be extended by
application developers, making it possible to specify things like e.g. mapping
fields to columns in a database.

To find out more about protocol buffers start at the [Protocol Buffers Wikipedia
Article](http://en.wikipedia.org/wiki/Protocol_Buffers) for some background.

## ORM

An object relational mapper can be used to store (nested) object structures in
a database. Every protobuf message is mapped to one or more tables. The exact
mapping depends on the type of fields contained in the message. Fields with a
primitive value type are mapped directly to columns in a table. Whereas
repeated data types as well as Message types are stored in separate tables
that reference the parent message by id from those tables.

## Protobuf Mapping Example

To get an idea of how a message is mapped to database tables, we start with a
simple message type named _Foo_ and then show how this is mapped to the
database.

The following code is stored in a file named "simple.proto".

```protobuf
package pb_orm_test;

import "orm.proto";

message Foo {
	optional string text = 1;
 	repeated int32 numbers = 2;
}
```

The message _Foo_ above contains both a _string_ field and a field consisting
of a collection of 32 bit integer numbers. The _string_ value will be stored
in the table of the _Foo_ message while the repeated field will be stored in a
separate table and then reference the _Foo_ message.

Note how the first line specifies a name for the package. The message _Foo_ will
be part of a namespace in C++ named after this package name.

Before the message is declared notice the ```import "orm.proto";``` statement.
The _orm.proto_ file contains extended field options. These options are used to
customize and direct the behavior of the orm code that will eventually store
instances of the _Foo_ message in the database.

After processing the "simple.proto" file containing the protobuf specification
above, the resulting C++ code consists of both accessor methods to create, read
and update _Foo_ messages in memory as well as a description class that allows
application code to reflect on the fields and data types of the message. The
header file generated for this code is named "simple.pb.h".

The generated protobuf description class is used by the ORM to generate SQL to
create tables on the fly. The ORM creates the following SQL for storing
instances of _Foo_:

```sql
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;
CREATE TABLE Foo_numbers (value INT,parent_id INTEGER,id INTEGER PRIMARY KEY AUTOINCREMENT);
CREATE TABLE Foo (text VARCHAR(255),id INTEGER PRIMARY KEY AUTOINCREMENT);
COMMIT;
```

The ORM has mapped the field ```optional string text = 1;``` of the message to a
```text VARCHAR(255)``` column in the _Foo_ table. The ```repeated int32 numbers
= 2;``` field has been mapped to a column ```value INT``` in separate table
*Foo_numbers*. *Foo_numbers* refers back to the _Foo_ table via ```parent_id
INTEGER```.

##Connecting to a Database

Before protobuf objects can be stored in the database, first a database must be
created. If a database already exists a connection to this database has to be
established.

```c++
#include "pb-orm.h"
...
OrmInitialize();
OrmConn conn = NULL;
if (OrmDatastoreSQLite3()) {
	OrmConnectSQLite3("./", "sample_db", conn);
}
```

The function call ```OrmInitialize();``` needs to be performed once for an
application at startup. After this call the application can create one or more
connections to the database. The function call ```OrmDatastoreSQLite3();```
checks the availability of the sqlite3 database library within the application.
The function call ```OrmConnectSQLite3("./", "sample_db", conn);``` will then
connect to or create a database in the current directory "./" named *sample_db*.

If the application is ready to terminate, the connection to the database should
be closed in order to flush any cashes and to free allocated memory.

```c++
#include "pb-orm.h"
...
OrmConnClose(conn);
OrmShutdown();
```

Here function call ```OrmConnClose(conn);``` will close a previously established
connection while ```OrmShutdown();``` will shutdown the orm correctly.
After this the application can terminate safely.

## Creating tables.

Now a database has been created and a connection to it has been established.
In order for _Foo_ messages to be stored in the database the tables for _Foo_ 
messages need to be created once.

```c++
#include "pb-orm.h"
#include "simple.pb.h"
...
OrmCreateTable(conn,::pb_orm_test::Foo::descriptor());
```

Note the package name *pb_orm_test* appearing here as the namespace of _Foo_.
Further note the class method call _Foo::descriptor()_ which returns an object
with type information that can be used by the ORM to create tables for storing
_Foo_ instances.

The ```OrmCreateTable``` function will use the type descriptor passed in and
create the tables _Foo_ and *Foo_numbers* as was described before.

So as mentioned before this only has to be done once after creating the
database.

If for some reason it is no longer desirable to store _Foo_ messages in the
database use the following code to destroy the tables.

```c++
#include "pb-orm.h"
#include "simple.pb.h"
...
OrmDropTable(conn,::pb_orm_test::Foo::descriptor());
```

This will only drop the tables for storing _Foo_ instances, not for any other
message types that may be stored in the database.

## Create, Read, Update, Delete (CRUD)

The four basic functions of persistent storage are _Create_, _Read_, _Update_
and _Delete_. This section describes how these operations can be performed using
orm.

### Create

Now that there is a valid connection to the database and the tables for _Foo_
have been created, inserting a new message into the database is easy.

```c++
#include "pb-orm.h"
#include "simple.pb.h"
...
::pb_orm_test::Foo msg;
pb::uint64 msgid;
if (!OrmMessageInsert(conn, msg, msgid)) {
	printf("error: could not insert message");
	return;
}
```

We created an empty message and inserted it into the database. The msgid
returned contains the numeric id of the newly inserted record.

### Read

We now should have exactly 1 message in the table. To find and read this message
use the following code.

```c++
if (!OrmMessageFind(conn, msg.descriptor(), msgid)) {
	printf("error: message not found");
	return;
}
	
OrmContext context;
OrmMessageRead(conn, msg, msgid, false, context);
```

The call to ```OrmMessageRead``` will read the message from the database. The
_OrmContext_ value that is returned in the variable _context_ can be used later
on to write modifications made to this message back into the database.

Note the 4th parameter which is set to _false_. This is the _recurse_ parameter
that determines whether content stored in other tables than the _Foo_ table will
be retrieved. In this case it means that the _repeated_ field _numbers_ will
**not** be retrieved from the database. So by setting _recurse_ to _false_ a
potentially large tree of nested objects is not read in all at once, but can be
retrieved incrementally.

### Update

To change the fields of the retrieved message we use the C++ accessors.

```c++
msg.set_text("hello, world!");
msg.add_numbers(123);
```

The code above will change the message in memory, now to save it to the database
use the following code:

```c++
OrmMessageUpdate(context);
OrmFreeContext(context);
```

The context knows about the msg instance that was retrieved and updated. Now
calling ```OrmMessageUpdate``` will take the contents of the message and write
it back into the database.

Note that the numbers field was not read from the database during
```OrmMessageRead``` because _recursive_ was _false_. However the number 123 was
added to the numbers, so what happens to any numbers previously assigned when
this new _numbers_ value is written to the database? The answer is that the
number will be added to the numbers already present so the existing content of
the collection is preserved.

If the context is no longer needed, it should be deallocated using a call to
function ```OrmFreeContext```.

### Delete

To delete a message from the database use the following code:

```c++
OrmMessageDelete(conn, msg.descriptor(), msgid);
```

