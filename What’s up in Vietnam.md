# What’s up in Vietnam
## Introduction
It's been 42 years since the end of The Vietnam War. Much has changed over the years. It’s still a country ruled by Communist Party with low GDP per capita. On the other hand, the external world forced to change the isolation policy to be more open and the situation is slowly evolving since the opening on market economy in 1986.

Is there a more sophisticated parallel to a part of computer science than situation in Vietnam? The first brave soul was Ted Neward with his article “The Vietnam of Computer Science” comparing Object Relational Mapping to one of the most disgraceful conflicts in U.S. history. It was written in 2006. 11 years have passed and many has changed in approach to ORM.

## What the Object Relational Mapping stand for?
Follow the definition on Wikipedia _it’s a technique for converting data between incompatible type systems using object-oriented programming languages._ To be stricter it’s converting relational data to object oriented domain model (OODM) and OODM to relational data. Conversions like that could be done in several ways, but the most common is usage of special tool called ORM library or ORM framework.

There is a bunch of ORM frameworks:
* the most known in Java world Hibernate - colossus with built in object oriented Hibernate Query Language, Unit of Work integrated;
* Entity Framework – most popular in .NET environment, with built in Entity SQL;
* Doctrine known in PHP world modeled on Hibernate;
* Active Record in Ruby world based on old C# framework, using Active Record pattern to access to persistence layer;
* Sequelize in Node.js world using same Active Record pattern;
* and many other presumably with one leading per technology.
ORM frameworks are available in the vast majority of applications with persistence layer implemented.

## Why is it problematic.
> Object systems are typically characterized by four basic components: identity, state, behavior and encapsulation.  
_Ted Neward_

Summing up the above thought we can say that every object is described by its identity independently of its state. Every object has a collection of operations that can be invoked to manipulate the state of itself or connected objects, and (most importantly) has encapsulated access to itself and effectively, the ability to inheritance and polymorphism.

> The representation of your object in memory depends what you intend to do with it, and context-sensitive representation is not a feature of OO design.  
_Laurie Voss_

On the other hand, relational systems describe the set of truths related with each other. Althought, thanks to the 4th generation language – SQL – we can easily retrieve the facts hidden behind relations, it’s not the most efficient way to store the state of collection of objects.

> If your project really does not need any relational data features, then ORM will work perfectly for you, but then you have a different problem: you're using the wrong data store. […] On the other hand, if your data is relational, then your object mapping will eventually break down.  
_Laurie Voss_

Let’s consider a situation where we want to create a self-driving PTCC. Vehicles can be depicted by a class Vehicle with some common properties and methods (turn left_right, brake), more precise classes (Bus, Tram, Train) can inherit from a Vehicle and introduce new properties and methods. More specific classes will inherit from Bus, Tram, Train etc. and introduce functionalities specific for bus_tram/train models. Every object has it’s own identity (for example a registration number) and state slightly different for every final class.

## There are three approaches to achieve object-relational mapping.

### Table-per-class approach
This is the normalized way to achieve mapping. Every single class map a separate table with properties. The  vehicle table is in the 1:n relation with buses_trams_trains. Tables buses_trams_trains are in 1:n relation with more specific classes. So, what’s the disadvantage? Try to manually create SQL to retrieve a record by the registration number, assuming we have about 50 models of vehicles. It’s absolutely impractical. We need to, in addition, create two artificial primary keys to determine records of more specific tables. Even if it happens behind the scenes, that’s a very inefficient way of retrievering data.

### Table-per-concrete-class approach
This approach has similar disadvantages. Let’s try to retrieve 10 of the oldest vehicles in our fleet. We need to union all data in really obscure way (creating many Null columns):
```
SELECT Col1, Col2, Null as Col3 FROM Model1,
UNION Col1, Null as Col2, Col3 FROM Model2.
```
… and then order by production date. It’s even more impractical and inefficient.

### Table-per-hierarchy approach
It’s the most denormalized approach, but almost all projects I know tend to turn towards it after performance tests. Of course, creating supertables (we have just one) are also bad solutions. Apart from obvious problems resulting from denormalization, we need to handle the database limitations (for example maximum number of columns for InnoDB engine of MySQL is 1017).

So why it’s problematic …

> Essentially what you are doing is synchronizing between two quite different representations of data.  
_Fowler._

Fowler is right about this. In the real world, a bus is not a relation for a Vehicle in real world. We’re bending reality, so that it conforms to the relational database.

## What changed and what are we yet to solve.
So, 11 years have passed.
1. Some problems have evolved. Joins are much faster now thanks to solutions like ClusterixDB. There are new built in mechanisms in relational databases. We can store documents in PostgreSQL database. Althought, it’s still not the best idea to create SQL with dozens of joins, but modern databases are optimized for less complex queries.
2. Over time, composition started winning over inheritance. Mapping composition to RDBS id easier thanks to its relational origin.
3. Using the same database by two applications is seen as bad practice. This and others policy problems mentioned by Ted Neward are a little outdated. But on the other hand, many services are starting cooperatively. Updating such services is a quite hard to achieve. There is a time where one service need to work with two versions of database. Forcing ORM framework to work with two different versions of database is almost impossible in clear and easy to achieve way.

> As a result, as the system grows over time, there will be increasing pressure on the developers to “tie off” the object model from the database schema, such that schema changes won’t require similar object model refactorings, and vice versa.  
_Ted Neward_

HQL, DQL and other object based languages evolved and become more powerful, but are still less felxible than plain SQL. 

> Even if 80% of users need only 30% of the features of SQL, then 100% of users have to break your abstraction to get the job done.  
_Laurie Voss_

4. This problem applies to the whole ORM concept. By escaping the abstraction of ORM anywhere in the code, we’re also breaking the abstraction of Unit of Work, losing transactions, calling for unnecessary data and making objects inside unit of work dirty without marking them as dirty. List of side effects of breaking ORM framework abstraction by using plain SQL is long and depends on specific framework and conditions.
5. Sometimes our objects are stored inside existing process in some kind of application cache. Database migration mechanism is often delivered with ORM framework. There is totally different situation with cache migrations systems. We need to manually solve such situations.
6. Usually, more powerful ORM frameworks have lazy loading mechanism. This feature is neither consistent nor obvious (usually implemented by proxy design pattern) and we need to be careful not to DDoS the database by an innocently looking iteration.
7. Many ORM frameworks incentive bad practices, like mixing persistence layer with domain layer, creating anemic domain objects, breaking Single Responsibility Principle. The worst are frameworks based on active records where connections are really strong. Trained software architects can avoid this and retain the strong separation between persistence  and domain model in exchange for painfull maintenance.

## Summary
As we can see, Vietnam is slowly changing, or to be stricter, it adjusts itself to external world. Is it enough to deal with it in current form? It depends. I’m not against ORM frameworks in general. There are places where we’re operating on relational data which can be simply mapped to object oriented world. Especially in small applications with domination of composition in the domain layer. What to do in other situations? Get to know and consider all limitations of usage of ORM framework and deal with it … or find other solution.

1. Use proper database to your need. Maybe it’s better to use document storage/key-value store (like in example with vehicles), graph database (in many social services) or other type of database.
2. Use CQRS, save events as known truths and visualize data in reporting stores in proper databases: relational, context dependent data in RDBS, objects in key-value stores, graphs in graph databases etc.
3. Use Half-way solution as resignation from ORM frameworks, but still usage RDBS. It’s recommended if we want to keep clear separation between domain and persistence layer without additional maintenance of persistence model needed for ORM framework. In exchange, we can use database with plain SQL more freely without the risk of side effects.
