---
title: "Transaction Functions"
weight: 85
menu:
  main:
    parent: manual
---


# Transaction functions


### Atomic processing within the transaction

Transaction functions 

- run on the transactor inside of transactions
- can atomically analyze and transform database values
- can perform arbitrary logic
- must have no side effect
- must return transaction data (`Future[Seq[Statement]]`)
- are available for Peers on the jvm platform only

Since tx functions have access to the tx database value they are essential to guaranteeing atomicity in updates for instance. You can query the current db value within the transaction logic and thus be sure certain assertions hold before doing some operation.

### Calling tx functions

Molecule facilitates writing tx functions by annotating one or more objects that contain tx function methods:

```scala
import molecule.macros.TxFns

@TxFns
object myTxFunctions {

  def myTxFunction1(args...)
    (implicit conn: Future[Conn], ec: ExecutionContext): Future[Seq[Statement]] = {
    ...
  }
  
  def myTxFunction2(args...)
    (implicit conn: Future[Conn], ec: ExecutionContext): Future[Seq[Statement]] = {
    ...
  }
  // etc...
}

@TxFns
object myTxFunctionsSomewhereElse {...}
// etc...
```
The macro annotation `@TxFns` creates an internal "twin" method at compile time for each Scala tx function that you define. The twin method basically adapts to the way Datomic expects tx functions and automatically gets saved into the Datomic database transparently without any work on your part.

#### Enabling the macro annotation for Scala 2.12

To use the `@TxFns` macro annotation with Scala 2.12, you'll need to include the Macro Paradise compiler plugin in your build:

```scala
libraryDependencies += compilerPlugin("org.scalamacros" % "paradise" % "2.1.1" cross CrossVersion.patch)
```

>Macro annotations have been considered experimental in Scala and for version 2.12 requires an
>import of the scala paradise plugin. In version 2.13 they have been incorporated into the core
>language itself and importing the paradise plugin is no longer necessary in 2.13.


### Tx functions on the transactor classpath

The tx functions also need to be visible to the transactor process meaning that the compiled tx function container must be on the classpath of the transactor. This will depend on the Datomic setup you are using:

#### Local dev / in-memory
No need to prepare anything since transaction functions defined within the project will be available on the classpath for the Transactor managed by the Peer.

#### Starter pro / pro
Set Datomic classpath variable to where your tx functions are before starting the transactor
```
> cd DATOMIC_HOME
> export DATOMIC_EXT_CLASSPATH=/Users/mg/molecule/molecule/coretests/target/scala-2.12/test-classes/
> bin/transactor ...
```

#### Free
The Free version can't set the classpath variable so we need to provide the tx functions manually by making a jar of our classes, move it to the transactor bin folder and start the transactor:
```
> cd ~/molecule/molecule/coretests/target/scala-2.12/test-classes  [path to your compiled classes] 
> jar cvf scala-fns.jar .
> mv ~/molecule/molecule/coretests/target/scala-2.12/test-classes/scala-fns.jar DATOMIC_HOME/bin/
> bin/transactor ...
```


### Tx function example

A typical example of needing access to the tx database value before doing an operation on it could be to transfer money from one account to another.

### Enforcing atomic constraints
We want to be sure that there is enough available funds before we do the transfer. If we had done the lookup of the current balance outside a transaction we could risk that the balance was updated inbetween our lookup and doing the transfer. With a tx function we can avoid this by having access to the tx database value _within the transaction itself_. This will give us the necessary atomicity of the whole transfer. The transfer will only happen if there are enough funds available.

Let's look at a simple tx function implementation:

```scala
@TxFns
object myTxFunctions {

  // Pass in entity ids of from/to accounts and the amount to be transferred
  def transfer(from: Long, to: Long, amount: Int)
              (implicit conn: Future[Conn], ec: ExecutionContext): Future[Seq[Statement]] = {
    for {
      // Validate sufficient funds in from-account
      curFromBalance <- Ns(from).int.get.map(_.headOption.getOrElse(0))

      _ = if (curFromBalance < amount)
      // Throw exception to abort the whole transaction
        throw TxFnException(
          s"Can't transfer $amount from account $from having a balance of only $curFromBalance."
        )

      // Calculate new balances
      newFromBalance = curFromBalance - amount
      newToBalance <- Ns(to).int.get.map(_.headOption.getOrElse(0) + amount)

      // Update accounts
      newFromStmts <- Ns(from).int(newFromBalance).getUpdateStmts
      newToStmts <- Ns(to).int(newToBalance).getUpdateStmts
    } yield newFromStmts ++ newToStmts
  }
}
```

The signature of a tx functions must include implicit `Conn` and `ExecutionContext` parameters. This makes sure that the macro annotation can inject a database value in the generated transparent twin function for Datomic.

We first check the available-funds constraint by looking up the current balance and throw an exception if there is not enough money available. Throwing an exception inside a tx function will cancel the whole transaction and thereby guarantee atomicity.

Tx functions need to return transaction statements. We use Molecule's tx methods on molecules for the equivalent operations to get the necessary statements. So, to get the transaction statements of an update we simply replace a normal `update` call too `getUpdateStmts` which will give us the transaction statements produced:

```scala
// Isolated update transaction (not in tx function)
Account(from).balance(newFromBalance).update

// .. equivalent to the transaction statements returned by `getUpdateStmts` 
Account(from).balance(newFromBalance).getUpdateStmts
```


### Tx functions have to be pure

Tx functions can't modify the database within the body of the tx method. You couldn't for instance do an update within the method body. Any operations on the database have to be encoded in the returned transaction statements.


### Invoking tx functions

We call the transaction function inside a `transactFn` method:
```scala
transactFn(transfer(fromAccount, toAccount, okAmount))
```
`transactFn` is a macro that needs the tx function invocation itself as its argument in order to be able to analyze the tx function at compile time.

So our complete example could look like this:

```scala
// Initial balances
Account(fromAccount).balance.get.map(_.head ==> 100)
Account(toAccount).balance.get.map(_.head ==> 700)

// Invoke tx function to do the transfer and pass the produced tx statements to `transactFn`
transactFn(transfer(fromAccount, toAccount, 20))

// Balances after transfer
Account(fromAccount).balance.get.map(_.head ==> 80)
Account(toAccount).balance.get.map(_.head ==> 720)
```

### Error handling - ensuring atomicity

```scala
for {
  fromAccount <- Ns.int(100).save.map(_.eid)
  toAccount <- Ns.int(700).save.map(_.eid)
  tooBigAmount = 200
  
  // Trying to transfer a too big amount will return a failed Future with an exception 
  _ <- transactFn(transfer(fromAccount, toAccount, tooBigAmount))
    .map(_ ==> "Unexpected success")
    .recover { case TxFnException(msg) =>
      msg ==> s"Can't transfer 200 from account $fromAccount having a balance of only 100."
    }
  
  // No data has been changed
  _ <- Account(fromAccount).balance.get.map(_.head ==> 100)
  _ <- Account(toAccount).balance.get.map(_.head ==> 700)
} yield ()
```


### Composing tx functions

A tx function can be called from another tx function. We can thus compose more complex tx functions from "sub tx functions". We could for instance de-compose our previous transfer tx function into two sub tx functions `withdraw` and `deposit` and then call those from a `transferComposed` tx function:

```scala
// "Sub" tx fn - can be used on its own or in other tx functions
def withdraw(from: Long, amount: Int)
            (implicit conn: Future[Conn], ec: ExecutionContext): Future[Seq[Statement]] = {
  for {
    curFromBalance <- Ns(from).int.get.map(_.headOption.getOrElse(0))

    _ = if (curFromBalance < amount)
      throw TxFnException(
        s"Can't transfer $amount from account $from having a balance of only $curFromBalance.")

    newFromBalance = curFromBalance - amount
    stmts <- Ns(from).int(newFromBalance).getUpdateStmts
  } yield stmts
}

// "Sub" tx fn - can be used on its own or in other tx functions
def deposit(to: Long, amount: Int)
           (implicit conn: Future[Conn], ec: ExecutionContext): Future[Seq[Statement]] = {
  for {
    newToBalance <- Ns(to).int.get.map(_.headOption.getOrElse(0) + amount)
    stmts <- Ns(to).int(newToBalance).getUpdateStmts
  } yield stmts
}

// Compose tx function by calling other tx functions
def transferComposed(from: Long, to: Long, amount: Int)
                    (implicit conn: Future[Conn], ec: ExecutionContext): Future[Seq[Statement]] = {
  for {
    // This tx function guarantees atomicity when calling multiple sub tx functions.
    // If they were called independently outside a tx function, atomicity wouldn't be guaranteed.
    withdrawStmts <- withdraw(from, amount)
    depositStmts <- deposit(to, amount)
  } yield withdrawStmts ++ depositStmts
}
```

### Adding tx meta data to tx function invocations

Tx meta data can be added to a tx function invocation by adding one or more tx meta data molecules with applied tx meta data to the `transactFn`/`transactFnAsync` method. Say we want to add meta information about "who did the transfer" then we can add it to the transaction entity like this:
```scala
// Add tx meta data that John did the transfer
transactFn(transfer(fromAccount, toAccount, 20), Person.name("John"))

// We can then query for the transfer that John did
Account(fromAccount).balance.Tx(Person.name_("John")).get.map(_.head ==> 80)
Account(toAccount).balance.Tx(Person.name_("John")).get.map(_.head ==> 720)
```
Note how the tx meta data applies to both accounts since they were both modified in the same transaction that the tx meta data was applied to.

We can add arbitrary and possibly unrelated tx meta data to a tx function invocation by applying two or more tx meta data molecules:
```scala
// Add tx meta data that John did the transfer and that it is a scheduled transfer
transactFn(
  transfer(fromAccount, toAccount, 20), 
  Person.name("John"), 
  UseCase.name("Scheduled transfer")
)

// Query multiple Tx meta data molecules
Account(fromAccount).balance
  .Tx(Person.name_("John"))
  .Tx(UseCase.name_("Scheduled transfer")).get.map(_.head ==> 80)

Account(toAccount).balance
  .Tx(Person.name_("John"))
  .Tx(UseCase.name_("Scheduled transfer")).get.map(_.head ==> 720)
```


### Inspecting tx function invocations

If you want to see the `Statement`s produced by a tx function you can invoke it within `inspectTransactFn` without affecting the live database:

```scala
// Print inspect info for tx function invocation
inspectTransactFn(transfer(fromAccount, toAccount, 20))

// Prints produced tx statements to output:
/*
## 1 ## TxReport 
========================================================================
ArrayBuffer(
  List(
    :db/add       17592186045445       :Account/balance    80        Card(1))
  List(
    :db/add       17592186045447       :Account/balance    720       Card(1)))
------------------------------------------------
List(
  added: true ,   t: 13194139534345,   e: 13194139534345,   a: 50,   v: Thu Nov 22 16:23:09 CET 2018

  added: true ,   t: 13194139534345,   e: 17592186045445,   a: 64,   v: 80
  added: false,  -t: 13194139534345,  -e: 17592186045445,  -a: 64,  -v: 100

  added: true ,   t: 13194139534345,   e: 17592186045447,   a: 64,   v: 720
  added: false,  -t: 13194139534345,  -e: 17592186045447,  -a: 64,  -v: 700)
========================================================================
*/
```
Two groups of data are shown. The first group is an internal representation in Molecule showing the operations. The second group shows the datoms produced in the transaction. For ease of reading, "-" (minus) is prepended the prefixes (t, e, a, v) for the datoms that are retractions - where `added` is false. The abbreviations represents the parts of the Datom:

- `added`: operation, can be true for asserted or false for retracted
- `t`: transaction entity id
- `e`: entity
- `a`: attribute
- `v`: value

Updating the from-account balance from 100 to 80 for instance creates two Datoms, one retracting the old value 100 and one asserting the new value 80.







### Next

[Time...](/manual/time)
