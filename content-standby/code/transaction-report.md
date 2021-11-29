---
title: "Transaction Report"
weight: 85
menu:
  main:
    parent: documentation
    identifier: code-transaction-report
---



# Transaction report




All assertions and retractions in Datomic happen within a transaction that guarantees ACID consistency. Along with the domain data involved, Datomic also automatically asserts a timestamp as part of a created Transaction entity for that transaction.

Say that we create a new person with entity id `e5`:
```scala
val e5 = Person.name("Fred").likes("pizza").save.eid
```

Then the following assertions are made:

![](/img/page/transactions/1.png)

The 4th column of the quintuplets is the entity id of the transaction where the fact was asserted or retracted. In this case the two facts about Fred was asserted in transaction `tx4`. `tx4` is an entity id (`Long`) exactly as the entity id of fred `e5`.

The time of the transaction `date1` (`java.util.Date`) is asserted with the transaction entity id `tx4` as its entity id. And since that timestamp fact is also part of the same transaction `tx4` is also the transaction value (4th column) for that fact.


### Transactions return TxReport

All transactional operations on molecules return a `TxReport` with information about the transaction like what data was transacted and what entities were created and a timestamp of the transaction:

```scala
val tx: TxReport = Person.name("Fred").likes("pizza").age(38).save

// Entity id created - useful when we know one entity was created
val fredId: Long = tx.eid // (same as tx.eids.head)

// Entities id created
val entities: Seq[Long] = tx.eids

// Transaction time `t` (a sequential number created internally by Datomic identifying the tx)
val txT: Date = tx.t

// Transaction time as `Date`
val txTime: Date = tx.inst

// Transaction entity id
val txEntityId: Long = tx.tx

// Transaction Entity
val txEntity: datomicEntity = tx.txE
```



### Transaction entity id `tx`

Since the transaction is itself an entity exactly as any other entity we can query it as we query our own domain data.

Molecule offers some generic attributes that makes it easy to access transaction data. In our example we could find the transaction entity id of the assertion of Freds `name` by adding the generic attribute `tx` right after the `name_` attribute (that we have made tacit with the underscore since we're not interested in value "Fred"):

_"In what transaction was Freds name asserted?"_
```scala
Person(e5).name_.tx.get.map(_.head ==> tx4  // 13194139534340L
```
The `tx` attribute gets the 4th quintuplet value of its preceeding attribute in a molecule. We can see that `name` of entity `e5` (Fred) was asserted in transaction `tx4` since the value `tx4` was saved as the `name` quintuplet's 4th value.

### Transaction value `t`

Alternatively we can get a transaction value `t`

```scala
Person(e5).name_.t.get.map(_.head ==> t4  // 1028L
```


### Transaction time `txInstant`

With the transaction entity available we can then also get to the value of the timestamp fact of that transaction entity. For convenience Molecule has a generic `txIntstant` to lookup the timestamp which is a `date.util.Date`:

_"When was Freds name asserted?"_
```scala
Person(e5).name_.txInstant.get.map(_.head ==> date1  // Tue Apr 26 18:35:41
```

### Transaction data per attribute

Since each fact of an entity could have been stated in different transactions, transaction data is always tied to only one attribute.

Say for instance that we assert Fred's `age` in another transaction `tx5`, then we could subsequently get the two transactions involved:

_"Was Fred's name and age asserted in the same transaction?"_
```scala
// Second transaction
val tx5 = Person(e5).age(38).update.tx

// Retrieve transactions of multiple attributes
Person(e5).name_.tx.age_.tx.get.map(_.head ==> (tx4, tx5)

// No, name and age were asserted in different transactions
tx4 !== tx5
```
Likewise we could ask

_"At what time was Fred's name and age asserted"_
```scala
Person(e5).name_.txInstant.age_.txInstant.get.map(_.head ==> (date1, date2)

// Fred's name was asserted before his age
date1.before(date2) === true
```