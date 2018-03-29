# What’s up in Vietnam
## Introduction
Is there a more sophisticated parallel to a part of computer science than a situation in Vietnam? The first brave soul was Ted Neward with his article “The Vietnam of Computer Science” comparing the Object Relational Mapping to one of the most disgraceful conflicts in the U.S. history. It was written in 2006. 12 years have passed, and many have changed in an approach to ORM.

## What the Object Relational Mapping stands for?
Follow the definition on Wikipedia _it’s a technique for converting data between incompatible type systems using object-oriented programming languages._ To be stricter it’s converting relational data to the object-oriented domain model (OODM) and OODM to relational data. Conversions like that could be done in several ways, but the most common is a usage of a tool called ORM library or ORM framework.

## Why is it problematic.
There are words such as „incompatible type systems” even in the definition, and all incompatibilities are a potential problem. My first question which appeared after getting into the subject was „Why the hell we are using incompatible systems? Why is it the first (and often the only one known) solution to solve the problem with persisting data?”

> If your project really does not need any relational data features, then ORM will work perfectly for you, but then you have a different problem: you're using the wrong data store. […] On the other hand, if your data is relational, then your object mapping will eventually break down.  
_Laurie Voss_

### Let’s characterize these two worlds

> Object systems are typically characterized by four basic components: identity, state, behavior and encapsulation.  
_Ted Neward_

Summing up the above thought, we can say that every object is described by its identity independently of its state. Every object has a collection of operations that can be invoked to manipulate the state of itself or connected objects, and (most importantly) has encapsulated access to itself and effectively, the ability to inheritance and polymorphism.

> The representation of your object in memory depends what you intend to do with it, and context-sensitive representation is not a feature of OO design.  
_Laurie Voss_

On the other hand, relational systems describe the set of truths related to each other. Although thanks to the 4th generation language – SQL – we can easily retrieve the facts hidden behind relations, it’s not the most efficient way to store the state of a collection of objects.

## How is it working under the hood?
There are two patterns that ORM frameworks use: **Active Records** and **Data Mapper**.

With Active Records, each model object inherits merely from a base Active Record object. 

Example:
```
kate = new User("Kate")
kate.save()
```

Base Active Record objects contain an implementation of all methods connected with persistence. It’s easily visible that Single Responsibility Principle is broken entirely here, but there are more problems with that. Such objects make tests more demanding and less intuitive. They spoil the common language. Kate is not able to save itself or delete itself in the real world. Objects ceased to be a model of reality and became a data containers. It’s much harder to achieve real encapsulation of such containers. 

Data Mapper, despite its lower popularity, seems to be a better choice. As we can guess the Data Mapper does not clutter our OODM. Instead of this, a separated service called EntityManager is mapping the state of our domain objects to a DML and in reverse.

Example:
```
kate = new User("Kate")
EntityManager::perist(kate)
```

Thanks that we can achieve quite clear domain and hide all dirt behind repositories, but just theoretically. All that process is a little bit magic for most developers. For example, there is a whole document about working with different fetching types in JPA in [this page](https://docs.jboss.org/hibernate/orm/3.6/reference/en-US/html/performance.html#performance-fetching-lazy). If we’re using Hibernate, those fetching types can be overridden by Hibernate specific types if JPA type is EAGER (more [in the documentation](https://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#fetching-fetch-annotation)). As a result we are fetching from repositories proxies, not real domain models in some situations, and domain models in others. Default settings are different in different ORM frameworks and different technologies. But it’s a fact that sometimes we gain proxies behind „the repository gate” and we need to remember about it and deal with it.

As we can see, the use of ORM almost always affects the whole application design. It already sounds awful that the whole application depends on one tool. However, the real hell is waiting behind „the repository gate”. 

## Real life problem.
Let’s imagine that we received an order from a client to create „an application to support the whole set of smart devices that can be connected together”. Every device has a common controller so the best model of the reality would be to create an abstract class `SmartDevice` and let other devices to inherit after this one. We can additionally categorize similar devices to not implement same methods twice, so we’ll create groups such as `LightManagementDevice`, `HeatingDevice`, `SecurityDevice` with common methods such as `callThePolice`. At the end of this path, we can create devices such as SmartLamp, SmartRobot, SmartCamera, etc. The number of types of devices to handle is around 120. Every instance of a device has a new unique ID and should be easily searchable by this ID.

Let’s try to solve such problem.

### Table-per-class approach
This is the normalized way to achieve mapping. Every single class map a separate table with properties. The `smart_devices` table is in the 1:n relation with `light_management_devices`/`heating_devices`/`security_devices`/`etc.` Tables `light_management_devices`/`heating_devices`/`security_devices` are in 1:n relation with more specific classes. So, what’s the disadvantage? Try to create SQL to retrieve a record by its ID manually. It’s impractical. We need to, besides, create two artificial primary keys to determine records of more specific tables. Even if it happens behind the scenes, that’s a very inefficient way of retrieving data.

### Table-per-concrete-class approach
This approach has similar disadvantages. Let’s try to retrieve ten last updated devices. We need to union all data in a really obscure way (creating many Null columns):
```
SELECT Col1, Col2, Null as Col3 FROM light_management_devices,
UNION Col1, Null as Col2, Col3 FROM heating_devices.
```
… and then order by production date. It’s even more impractical and inefficient.

### Table-per-hierarchy approach
It’s the most denormalized approach, but almost all projects I know tend to turn towards it after performance tests. Of course, creating super-tables (we have just one) are also bad solutions. Apart from obvious problems resulting from denormalization, we need to handle the database limitations (for example the maximum number of columns for InnoDB engine of MySQL is 1017). It’s less than nine specific properties per single device in our case. Besides, our application becomes unscalable.

There is no right way to achieve it. Every approach has a lot of limitations.

> Essentially what you are doing is synchronizing between two quite different representations of data.  
_Fowler._

Fowler is right about this. A camera is not a relation to a smart device in the real world. We’re bending reality so that it conforms to the relational database.

## Is there any good solution to our problem?
Yes, we defined it in the first sentence of the definition of our problem. It is actually even not our solution. Our customer gave them to us. He wanted to create „an application to support the whole **set** of smart devices”. It means that we want to store a set of some data of the whole set of smart devices. And what’s the easiest way to save a set of data? Find a database that supports sets. There is a lot of them: AWS DynamoDB, InfinityDB, Redis, and others. Such data set can be easily mapped to documents too. And there are waiting even more promising databases such as MongoDB or document store built in PostgreSQL.

There are a lot of other databases matching different requirements. There are useful graph databases such as Neo4j (excellent database), column stores, spatial databases, probabilistic databases and many others.

## „OK, but I have relational data for sure and I need to use OO design of application because of some reason.”
I will tell you a story.

Once I had a task to create a wholly complicated ranking. Ranking should be working live for 27 000 active users. Every activity of the user should be reflected in scoreboard live. My first thought was to create some simplified CQRS with a system such as Apache Kafka calculating the ranking based on the stream of events.

Then my tech leader advised me: „don’t do it, use Sphinx”. I was totally lost, because Sphinx is a full-text search engine. How the hell deal with it to create ranking? Every row should have a full-text column. What to do with it? „Just leave it empty”, he said.

In the meantime, our team went to Neo4j conference which was organized by CD PROJECT company. They were creating a significant online card came - Gwent. I realized that they have to have even more significant problems with ranking, so I asked how they deal with it. „We’re using Sphinx” they answered. Then my tech lead asked them: „Why Sphinx? It’s a full-text search engine. How do you deal with a mandatory full-text field in a row?”. „Yeah, but it’s the simplest way to achieve it, and we just leave this mandatory field empty” they answered. I fooled myself.

This text is a bit naive too. I’m not forcing you to use more difficult tool to achieve a kind of karma. Relational databases and ORMs are advanced tools and I’m using it often privately. Just think at first a little bit if chosen technology fits your requirements. Relational databases and ORMs are not the only right way, it shouldn’t be even default way. It should be a decision.

## So … what changed since Vietnam War and what are we yet to solve in ORMs?
So, 11 years have passed.
1. Some problems have evolved. Joins are much faster now thanks to solutions like ClusterixDB. There are newly built-in mechanisms in relational databases. We can store documents in PostgreSQL database. Although it’s still not the best idea to create SQL with dozens of joins, but modern databases are optimized for less complex queries.
2. Over time, composition started winning over inheritance in OOP. Mapping composition to RDBS is easier thanks to its relational origin.
3. Using the same database by two applications is seen as a bad practice. This and others policy problems mentioned by Ted Neward are a little outdated. But on the other hand, many services are starting cooperatively. Updating such services is a quite hard to achieve. There is a time where one service needs to work with two versions of a database. Forcing ORM framework to work with two different versions of the database is almost impossible in clear and easy to achieve way.

> As a result, as the system grows over time, there will be increasing pressure on the developers to “tie off” the object model from the database schema, such that schema changes won’t require similar object model refactorings, and vice versa.  
_Ted Neward_

4. HQL, DQL, and other object-based languages evolved and became more powerful, but are still less flexible than plain SQL. 

> Even if 80% of users need only 30% of the features of SQL, then 100% of users have to break your abstraction to get the job done.  
_Laurie Voss_

This problem applies to the whole ORM concept. By escaping the abstraction of ORM anywhere in the code, we’re also breaking the abstraction of Unit of Work, losing transactions, calling for unnecessary data and making objects inside unit of work dirty without marking them as dirty. List of side effects of breaking ORM framework abstraction by using plain SQL is long and depends on specific framework and conditions.
5. Sometimes our objects are stored inside an existing process in kind of application cache. Database migration mechanism is often delivered with ORM framework. There is an entirely different situation with cache migrations systems. We need to solve such situations manually.
6. Usually, more powerful ORM frameworks have lazy loading mechanism. This feature is neither consistent nor obvious (traditionally implemented by proxy design pattern), and we need to be careful not to DDoS the database by an innocently looking iteration.
7. ORM frameworks, which using Active Record pattern incentive bad practices, like mixing persistence layer with domain layer, creating anemic domain objects, breaking Single Responsibility Principle. Trained software architects can avoid this and retain the sharp separation between persistence and domain model in exchange for painful maintenance.

## Summary
As we can see, Vietnam is slowly changing, or to be stricter; it adjusts itself to the external world. Is it enough to deal with it in current form? It depends. I’m not against ORM frameworks in general. There are places where we’re operating on relational data which can be mapped merely to object-oriented world. Especially in small applications with a domination of composition in the domain layer. What to do in other situations? Get to know and consider all limitations of usage of ORM framework and deal with it … or find other solution.

1. Use a proper database to your need. Maybe it’s better to use document storage/key-value store (like in the example with smart devices), graph database (in many social services) or other types of database.
2. Use CQRS, save events as known truths and visualize data in reporting stores in proper databases: relational, context-dependent data in RDBS, objects in key-value stores, graphs in graph databases etc.
3. Use Half-way solution as a resignation from ORM frameworks, but still usage RDBS. It’s recommended if we want to keep the clear separation between domain and persistence layer without additional maintenance of persistence model needed for ORM framework. In exchange, we can use the database with plain SQL more freely without the risk of side effects.

But, what is the most important: make the wholesome **decision**. The choice of technology used cannot be a coincidence, trends or just default choice same for every problem.
