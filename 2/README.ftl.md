
# Step 2: Storing Menu Data Using MongoDB

The Yummy Noodle Bar application core has been implemented, and you are tasked with extending it to store data.

We work within the Persistence domain to add this functionality. (life presever pic).

In that domain we have a representation of MenuItem optimised for persistence.

There is an event handler `MenuPersistenceEventHandler`, exchanging events with the application core,
and the repository `MenuItemRepository` whose responsibility is to persist and retrieve MenuItem data for the rest of the application.

You will implement `MenuItemRepository` using Spring Data MongoDB and integrate this with the `MenuPersistenceEventHandler`

## About MongoDB

MongoDB is a document oriented database that stores data natively in a document format called BSON (Binary JSON). This is similar in structure to JSON, and is ultimately derived from it.

It does not enforce a schema or document structure beyond the concept of the *Collection*, which contains a set of documents.

Querying can be performed over the whole document structure, although Joins between Collections is not part of the document model.

Large scale data transformation and analysis is a native part of MongoDB, and can be performed either declaratively, or via JavaScript functions as part of a Map Reduce implementation.

## Install MongoDB

Before continuing, ensure that you have MongoDB installed correctly.

If you don't have it installed already, visit the [Mongo DB Project](http://www.mongodb.org) and follow the instructions to install MongoDB.  

Do not set up any authentication.

After you have installed it, launch it:

```sh
$ mongod
```

You should see something like this:

```
all output going to: /usr/local/var/log/mongodb/mongo.log
```

Now launch the command line client:

```sh
$ mongo
```
 
And see the response:

```
MongoDB shell version: 2.0.8
connecting to: test
>
```

## Import Spring Data MongoDB

Import spring-data-mongodb into your project by adding it to your `build.gradle` list of dependencies:

    <@snippet "build.gradle" "deps" />

## Start with a (failing) test, introducing MongoTemplate

Before we can implement the Repository, we have to have something to use with it, the Domain, in this case meaning persistence, class.

Create a new test class `com.yummynoodlebar.persistence.integration.MenuItemMappingIntegrationTests`

This class will test that `com.yummynoodlebar.persistence.domain.MenuItem` works as expected against a real, running MongoDB instance.

Before we can pass that test, however, we need some tools to test with.

`MongoTemplate` is on of the core classes provided by Spring Data MongoDB.  It follows the familiar template pattern that is used extensively in other parts of Spring,
such as `JmsTemplate`, `JdbcTemplate` and `RestTemplate`.

It allows quick and easy connection to a MongoDB server and exposes a large amount of functionality in a single place.

Test Driven Development guides us to test the smallest piece of the system we can, and build our tests outwards from that; building confidence in the system as a whole.

The smallest piece in this case is the domain class, MenuItem.  It will contain mapping and configuration information describing how it should be persisted into the database.

There are two existing helper classes, `com.yummynoodlebar.persistence.domain.fixture.PersistenceFixture` and `com.yummynoodlebar.persistence.domain.fixture.MongoAssertions`
They provide some methods we can use to make our tests a bit more readable.

Update `MenuItemMappingIntegrationTests` to read:

    <@snippet "src/test/java/com/yummynoodlebar/persistence/integration/MenuItemMappingIntegrationTests.java" "top" />

This is a simple usage of MongoTemplate, using the persistence.domain.MenuItem class to push data into and out of a Mongo Collection.

TODO, describe the connection process - localhost, no security, default port 27017

Here, the test ensures that the mapping works as expected, and the document appears in the expected shape in the collection.  It also tests that the indexes that we expect
have also been initialised.

Run this test, it will fail, as the mapping is not as expected, the collection being used is wrong, and the indexes are not being fully created
We can now move onto altering our domain class to ensure it persists correctly.

Open `com.yummynoodlebar.persistence.domain.MenuItem`.

Add the annotations @Document, @Id and @Indexed to bring the domain into line with the test expectations.

    <@snippet "src/main/java/com/yummynoodlebar/persistence/domain/MenuItem.java" "top" />

This alters the collection used to be *menu* (instead of *MenuItem*), ensures that the field *id* is used as the Mongo document *_id* and that the field *name* is stored as *itemName* and is also indexed.

None of these are annotations are necessary, a bare POJO (Plain Old Java Object) can be passed to MongoTemplate and it will apply its default behaviour.  We have chosen to alter that behaviour using these mapping annotations on the persistence entity.

## Implement a CRUD repository

MenuItem is now ready to persist.  We could write an implementation of `MenuItemRepository` using MongoTemplate. Many applications do this successfully. But Spring Data gives us another option.  It can create an implementation of the Repository interface for us at runtime.

To take advantage of this, we first need to update `MenuItemRepository` into something that Spring Data can handle.

Before we do that though, a test!

Create a new class at `com.yummynoodlebar.config.MongoConfiguration`, leaving it empty for now.

Then create a test like so:

    <@snippet "src/test/java/com/yummynoodlebar/persistence/integration/MenuItemRepositoryIntegrationTests.java" />

This will fail.

Open `com.yummynoodlebar.persistence.repository.MenuItemRepository` and make it look like this:

    <@snippet "src/main/java/com/yummynoodlebar/persistence/repository/MenuItemRepository.java" />

CrudRepository is part of the Spring Data repository hierarchy. It acts as both a marker interface and it adds several key methods to provide us with a living, breathing implementations of a repository.

Everything will still compile in the project, however the test will not pass.

Open `MongoConfiguration` again, and alter it to read like so:

    <@snippet "src/main/java/com/yummynoodlebar/config/MongoConfiguration.java" />

`@Configuration` marks the class as a Spring Configuration/Java Config class, able to generate part of a Spring ApplicationContext.
`@EnableMongoRepositories` is part of Spring Data, and works to construct the repository implementation discussed earlier. The values passed into the annotation
select the class(es) that we want to be considered.  In this case, only `MenuItemRepository` is to be considered, and so it is referred to explicitly.

The guts of the configuration deal with connecting to MongoDB.  Creating a Mongo driver connection and a MongoTemplate to wrap it.

The auto generated Repositories use the MongoTemplate created here to connect to MongoDB.

Run the test again, it should now pass cleanly.

Congratulations!  You have a working repository, without having to actually implement it yourself.

It only does CRUD. Is that enough?

## Extend the Repository with a Custom Finder

A late breaking requirement has been uncovered!

Users are going to be given the opportunity to select dishes by the names of the ingredients that they contain.

Looking at MenuItem, the document that is stored in MongoDB will look similar to

```json
{
        "_id" : ObjectId("520d388bea7e3adc2a054886"),
        "_class" : "com.yummynoodlebar.persistence.domain.MenuItem",
        "ingredients" : [
                {
                        "name" : "Noodles",
                        "description" : "Crisp, lovely noodles"
                },
                {
                        "name" : "Peanuts",
                        "description" : "A Nut"
                }
        ],
        "cost" : "12.99",
        "minutesToPrepare" : 0
}
```

This will require querying deep inside the document structure to correctly identify the matching documents.

A test.

    <@snippet "src/test/java/com/yummynoodlebar/persistence/integration/MenuItemRepositoryFindByIngredientsIntegrationTests.java" />

This assumes a method named `findByIngredientsNameIn` on the repository. How do we implement that? Spring Data does it for us! We already have the method signature in the interface.

`standardItem()` has peanuts, `eggFriedRice()` doesn't.  So we should see just two documents come back from the query.

Try running the test....  it passes, right?

Spring Data has generated an implementation of this method, doing what we wanted. There is a rich vocabulary that you can express using the method names, see more in the [Reference Documentation](http://static.springsource.org/spring-data/data-mongodb/docs/current/reference/html/mongo.repositories.html#mongodb.repositories.queries)

## Extend the Repository with Map/Reduce

A more esoteric requirement would be helping user look up the ingredients used in the most dishes.

MongoDB provides a system to perform this kind of analysis, Map/Reduce(/Filter).

To use Map/Reduce, we need to gain access to the MongoTemplate directly and add this into the Repository that Spring Data is currently managing.

Create a new interface `com.yummynoodlebar.persistence.repository.AnalyseIngredients`:

    <@snippet "src/main/java/com/yummynoodlebar/persistence/repository/AnalyseIngredients.java" />

Next, update `MenuItemRepository` to include the `AnalyseIngredients` interface. This indicates to Spring Data that it should look for an implementation of that interface for extension.

We can now write an implementation of this new interface. Before doing that though, you must write a test for the new behaviour.

Create a test class `com.yummynoodlebar.persistence.integration.MenuItemRepositoryAnalyseIngredientsIntegrationTests`

    <@snippet "src/test/java/com/yummynoodlebar/persistence/integration/MenuItemRepositoryAnalyseIngredientsIntegrationTests.java" />

This sets up some known test data and calls the analysis method, expecting certain ingredients in known relative quantities.

Lastly, create an implementation of this interface `com.yummynoodlebar.persistence.repository.MenuItemRepositoryImpl`. The name of this class `MenuItemRepositoryImpl` is very important! This marks it out as an *extension* of the repository named `MenuItemRepository`, and is automatically component scanned, instantiated and used as a delegate by Spring Data.

    <@snippet "src/main/java/com/yummynoodlebar/persistence/repository/MenuItemRepositoryImpl.java" />

This class references two javascript functions, the mapper and the reducer, respectively.

Create 2 new javascript files, in the src/main/resources directory:

    <@snippet "src/main/resources/ingredientsmap.js" />

    <@snippet "src/main/resources/ingredientsreduce.js" />

The test should now pass successfully.

Congratulations! You have created a custom data analysis task against MongoDB.

## Summary

Now that the Menu data is safely stored in Mongo, its time to turn your attention to the core of the system, Orders

[Next… Storing Order Data using JPA](../3/)