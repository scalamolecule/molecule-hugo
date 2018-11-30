---
date: 2015-01-02T22:06:44+01:00
title: "Parameterized"
weight: 60
menu:
  main:
    parent: attributes
up: /manual/attributes
prev: /manual/attributes/aggregates
next: /manual/entities
down: /manual/entities
---

# Parameterized Input-molecules

Tests: 
[1 input](https://github.com/scalamolecule/molecule/blob/master/coretests/src/test/scala/molecule/coretests/input1),
[2 inputs](https://github.com/scalamolecule/molecule/blob/master/coretests/src/test/scala/molecule/coretests/input2),
[3 inputs](https://github.com/scalamolecule/molecule/blob/master/coretests/src/test/scala/molecule/coretests/input3)


Molecules can be parameterized by applying the input placeholder `?` as a value to an attribute. The molecule then expects input for that
attribute at runtime.

By assigning parameterized "Input-molecules" to variables we can re-use those variables to query for 
similar data structures where only some data part varies:

```scala
// 1 input parameter
val person = m(Person.name(?))

val john = person("John").get.head
val lisa = person("Lisa").get.head
```

Of course more complex molecules would benefit even more from this approach.

### Datomic cache and optimization
Datomic caches and optimizes queries from input molecules so performance-wise it's a good idea to use them.


### Parameterized expressions

```scala
val personName  = m(Person.name(?))
val johnOrLisas = personName("John" or "Lisa").get // OR
```

### Multiple parameters
Molecules can have up to 3 `?` placeholder parameters.

```scala
val person      = m(Person.name(?).age(?))
val john        = person("John" and 42).get.head // AND
val johnOrJonas = person(("John" and 42) or ("Jonas" and 38)).get // AND/OR
```

### Mix parameterized and static expressions

```scala
val americansYoungerThan = m(Person.name.age.<(?).Country.name("USA"))
val americanKids         = americansYoungerThan(13).get
val americanBabies       = americansYoungerThan(1).get
```

For more examples, please see the 
[Seattle examples](https://github.com/scalamolecule/molecule/blob/master/examples/src/test/scala/molecule/examples/seattle/SeattleTests.scala#L136-L233)
and tests for [1 input](https://github.com/scalamolecule/molecule/blob/master/coretests/src/test/scala/molecule/coretests/input1),
              [2 inputs](https://github.com/scalamolecule/molecule/blob/master/coretests/src/test/scala/molecule/coretests/input2),
              [3 inputs](https://github.com/scalamolecule/molecule/blob/master/coretests/src/test/scala/molecule/coretests/input3)



### Next

[Entities...](/manual/entities)