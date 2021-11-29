---
title: "Create db"
weight: 50
menu:
  main:
    parent: setup
---

# Schema transaction code

To create Datomic databases, Molecule generates a schema definition file from our Data Model that we can transact in Datomic to create the database in Datomic:

```
object SeattleSchema extends SchemaTransaction {
  
  lazy val partitions = Util.list()

  lazy val namespaces = Util.list(
    
    // Community --------------------------------------------------------

    Util.map(":db/ident"             , ":Community/name",
             ":db/valueType"         , ":db.type/string",
             ":db/cardinality"       , ":db.cardinality/one",
             ":db/fulltext"          , true.asInstanceOf[Object],
             ":db/doc"               , "A community's name",
             ":db/index"             , true.asInstanceOf[Object]),
    
    Util.map(":db/ident"             , ":Community/url",
             ":db/valueType"         , ":db.type/string",
             ":db/cardinality"       , ":db.cardinality/one",
             ":db/doc"               , "A community's url",
             ":db/index"             , true.asInstanceOf[Object]),
             
    // etc...
}
```
As you see, the `Community` namespace information is present in the value of the first pair in the map for the `name` attribute:

```
":db/ident", ":Community/name",
```
The rest of the lines are pretty self-describing except from the last two that create and save the internal id of the attribute in Datomic. [Datomic schemas](https://docs.datomic.com/on-prem/schema.html) are literally a set of datoms that have been transacted as any other data.


### Partition transaction data

Partition transaction data looks almost like attribute transaction data:

```
lazy val partitions = Util.list(

Util.map(":db/ident"             , ":gen",
         ":db/id"                , Peer.tempid(":db.part/db"),
         ":db.install/_partition", ":db.part/db"),
```
... except that the information is now installed internally in Datomic as partition data instead of as attribute data.

Partition examples with Molecule:

- [partitioned schema definition](https://github.com/scalamolecule/molecule/blob/master/coretests/src/main/scala/molecule/coretests/schemaDef/schema/PartitionTestDefinition.scala)
- [partition tests](https://github.com/scalamolecule/molecule/blob/master/molecule-tests/src/test/scala/molecule/tests/core/schemaDef/partition.scala)

More about [partitions in Datomic](https://docs.datomic.com/on-prem/indexes.html#partitions).



### External schema (lowercase) + external data (lowercase)

If both external schema and data is created with lowercase namespace names, then you can transact uppercase attribute aliases with the live database so that it will recognize your uppercase molecule code ([example](https://github.com/scalamolecule/molecule/blob/master/examples/src/test/scala/molecule/examples/mbrainz/MBrainz.scala#L38)):

```scala
conn.datomicConn.transact(MBrainzSchemaLowerToUpper.namespaces)
```







### Create new Datomic database

Now we can simply pass the generated raw schema transaction data to Datomic in order to create our database:

```scala
implicit val conn = recreateDbFrom(SeattleSchema)
```

The returned connection to the database is saved in an implicit val that allows us to create and operate on molecules in the following code.



## Managing databases

Datomic databases are created with a database name so that we can later refer to a specific database. In the above creation example, a random database name was created which is convenient for testing purposes.

For durable databases we use a database name:

```scala
// Create new database with identifier
implicit val conn = recreateDbFrom(SeattleSchema, "myDatabase")
```
Then we can later - in another scope - establish a new connection to the existing database:

```scala
// Create connection to the database 'myDatabase' 
implicit val conn = molecule.facade.Conn("myDatabase")
```

### Protocols

We can also supply a protocol like 'mem' for in-memory db, or 'dev' for a development db saved on local disk etc.

```scala
// Create new database with identifier as an in-memory database
implicit val conn = recreateDbFrom(SeattleSchema, "myDatabase", "mem")
```


### Working with non-molecule Datomic databases

If you are working with externally defined Datomic databases or data sets with lowercase namespace names defined then you can easily add some attribute name aliases so that you can freely work with the external data from your molecule code.

The sbt-plugin generates two additional schema transaction files that can be transacted with the external lowercase database so that you can use your uppercase Molecule code with it:

### Molecule schema (uppercase) + external data (lowercase)

When importing external data ([example](https://github.com/scalamolecule/molecule/blob/master/examples/src/test/scala/molecule/examples/seattle/SeattleTests.scala#L367-L368)) from a database with lowercase namespace names then you can transact lowercase attribute aliases ([example](https://github.com/scalamolecule/molecule/blob/master/examples/src/test/scala/molecule/examples/seattle/SeattleSpec.scala#L18)) so that your uppercase Molecule code can recognize the imported lowercase data:

```scala
conn.datomicConn.transact(SchemaUpperToLower.namespaces)
```











### Next

See how we can model our domain with attributes in [Schema](/documentation/schema/)...
