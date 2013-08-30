 
# Storing the Order Status in GemFire using Spring Data GemFire

In the Yummy Noodle Bar application, the statuses of Orders will be stored in GemFire.
These statuses will be coming into the application from the kitchen and order processing side of the business, as opposed to the orders themselves that will come from the system that accepts orders from clients.

## About GemFire

GemFire is a high performance distributed data grid.  It scales from a small embedded cache implementation to large scale wide area network implementations with data residency and access control.

Spring Data allows the creation of both server and client connections, data access, caching and deep integration with the Spring Application Context.

You will see here the creation of a Spring Data interface to a GemFire server. The next section of the tutorial shows the extension of this functionality to cover the use of Continuous Queries and how to integrate those into the Yummy Noodle application.

## Import Spring Data GemFire

In build.gradle, add the following to your list of repositories:

    <@snippet "build.gradle" "libs" />

Add the following to your list of dependencies:

    <@snippet "build.gradle" "deps" />

This Maven-style repository is required to access the GemFire libraries, which are not available from maven central.

## Run a GemFire Cache Server

In order to run the tests and perform GemFire development, it is necessary to have access to a GemFire server.

While it would be possible to download a full distribution, configure and run that, for the purposes of this tutorial, it is preferable to set up a server within this project.

In build.gradle, add this to the end of the file:

    <@snippet "build.gradle" "run" />

Create a new XML file in src/main/resources/server:

    <@snippet "src/main/resources/server/cache-config.xml" />

This configures a basic GemFire server, and creates a *Region*, a logical partition within GemFire, that we have named 'YummyNoodleOrder'.

Lastly, create the driving class `com.yummynoodlebar.persistence.services.LocalGemfireServer`:

    <@snippet "src/main/java/com/yummynoodlebar/persistence/services/LocalGemfireServer.java" />

You may now start a GemFire server (on port 40404) by running 

    ./gradlew run

This server will have access to the classpath of the project, most notably the OrderStatus class.  It is necessary for the GemFire server to have access to this class if we want to persist it within the grid.  When you create a standalone GemFire grid, you will need to provide any classes you wish to persist within a jar file on the classpath of every GemFire server.

This tutorial does not show how to set up an embedded GemFire node.  While it would be possible to run much of this section of the tutorial using an embedded node, the next section requires the use of a separate GemFire server.

## Start with a (failing) test, introducing GemFire Template

In a similar way as with MongoDB and JPA, the first test you will write is to check that OrderStatus can be correctly persisted into GemFire.

First, create a new empty class `com.yummynoodlebar.config.GemfireConfiguration`. This is a placeholder for the configuration until we have written a test.

Create a new test `com.yummynoodlebar.persistence.integration.OrderStatusMappingIntegrationTests` with the content.

    <@snippet "src/test/java/com/yummynoodlebar/persistence/integration/OrderStatusMappingIntegrationTests.java" />

This test uses `GemFireTemplate`, seen in this test via its API interface `GemFireOperations`.  This follows the same pattern as other Spring Template classes, exposing the most common operations using consistent, simple methods, and also providing access to the low level GemFire API in a managed way via callbacks.

You can see the access to the low level GemFire API in the clear() method.  This accesses the Region instance and clears it of all data.  Region implements the Map interface, as it is also conceptually a Map.  GemFire provides many features around this core concept, but you can see the map usage in the test method itself:

    <@snippet "src/test/java/com/yummynoodlebar/persistence/integration/OrderStatusMappingIntegrationTests.java" "yummyTemplatePut" />

GemFireTemplate exposes a Map oriented method to interact with its configured region.

The test inserts a single OrderStatus into the GemFire Region and then performs a query, using the GemFire Object Query Language (OQL).  This is a declarative language conceptually similar to the JPA Query Language/ Hibernate Query Language, providing a syntax to query against a set of Objects and their properties and perform selections, ordering, grouping and projections against the results.

Now that a test is in place, implement GemFireConfiguration with the content:

    <@snippet "src/main/java/com/yummynoodlebar/config/GemfireConfiguration.java" />

This class is mainly used, at the moment, to allow the consistent use of Spring Java Configuration in tests and other context creation.  It will be extended below and in the next tutorial section.

Currently, the important line is:

    <@snippet "src/main/java/com/yummynoodlebar/config/GemfireConfiguration.java" "import" />

This imports a traditional XML based Spring configuration.  Currently, Spring Data GemFire is significantly easier to configure using XML, and certain features are not yet fully implemented.  For this reason, XML configuration is still recommended for Spring Data GemFire.

Create a new file `src/main/resources/gemfire/client.xml`:

    <@snippet "src/main/resources/gemfire/client.xml" />

This configuration uses the GemFire Spring configuration namespace.

Firstly, it creates a GemFire DataSource.  This is a connection to a GemFire data grid. In this case, connecting to the GemFire server running on localhost:40404, which the local server configured above will run on.

The bean `yummyTemplate` is the instance of GemFireTemplate that is used in the test above. It is set up to communicate with a specific GemFire Region, YummyNoodleOrder, which must exist in the server. Again, this is configured in the server above.

The last two configurations that set up the GemFire transactional behaviour and integrate it with the Spring Transaction management system.


## Implement a CRUD repository

You have seen the creation of two Repository implementations against both MongoDB and JPA.  The process for creating a Spring Data GemFire Repository is consistent with the others.

First, create a new test `OrderStatusRepositoryIntegrationTests`:

    <@snippet "src/test/java/com/yummynoodlebar/persistence/integration/OrderStatusRepositoryIntegrationTests.java" />

This test generates a new OrderStatus with a known key and passes it to OrderStatusRepository for persisting. It then retrieves the data using the method `findOne`, which will query against the *key* that is passed into the GemFire Region Map structure.

Note that data is being managed explicitly in the test, rather than using the declarative transaction management that was introduced in the JPA tests.  While GemFire does integrate with the Spring provided transactions, it only support Isolation.READ_COMMITTED.  This means that once you write data, it cannot be read, by any thread or process, until the surrounding transaction is committed.  Any test that wrote data within a transaction would be unable to read it until the transaction finished.

For this reason, the test is not marked as @Transactional, so all data access will not be transactionally managed within the tests.  At the start and end of the test, the region is purged by using the repository deleteAll method generated by Spring Data.

To implement the Repository, update `OrderStatusRepository` to read:

    <@snippet "src/main/java/com/yummynoodlebar/persistence/repository/OrderStatusRepository.java" />

Update `GemFireConfiguration` to enable GemFire Repositories:

    <@snippet "src/main/java/com/yummynoodlebar/config/GemfireConfiguration.java" />


As with the other data stores, explicitly choose the Repository interface for Spring Data GemFire to implement.

As with the other data stores, the Entity/persistence class, in this case OrderStatus, requires annotating to control how it is persisted into the data store.

GemFire was built from the beginning to understand Java objects more thoroughly than either MongoDB or H2. This means that OrderStatus requires no additions to persist naturally into GemFire, as we saw above in `OrderStatusMappingIntegrationTests`.

Two things are necessary, however for Spring Data to be able to generate a Repository implementation, configuring the default Region, and specifying the property to use as the Region key/ID.

    <@snippet "src/main/java/com/yummynoodlebar/persistence/domain/OrderStatus.java" "gemfire" />

First, run the local GemFire server

    ./gradlew run
    
Then run the test `OrderStatusRepositoryIntegrationTests` to check that the OrderStatusRepository is being correctly generated and works as expected.

## Extend the Repository with a Custom Finder

An Order requires a history of the status updates made to it. A history is a list of OrderStatus in date order.

This requires a more complex query than simply by ID or Order ID. It will also require a sort by date. 

Create a new test `OrderStatusGetHistoryIntegrationTests`:

    <@snippet "src/test/java/com/yummynoodlebar/persistence/integration/OrderStatusGetHistoryIntegrationTests.java" />

This test creates a sequential history of a single order id, saves that list into GemFire, and then retrieves it using a new custom method.

Update the repository to read:

    <@snippet "src/main/java/com/yummynoodlebar/persistence/repository/OrderStatusRepository.java" />

This looks similar to the JPA custom method, and the concept is the same.  Create a new method and annotate it with a @Query, passing a string containing OQL to perform the query with.
This query selects the distinct elements from the YummyNoodleBar Region where the order is given and then orders by statusDate, which is a property on OrderStatus.

This will pass, with the correct ordering of the history, ordered by status date.

## Summary

Congratulations, Order Status data is safely stored in GemFire.

Next, you will learn to take advantage of GemFire Continuous Queries to extend the scalable, event driven architecture to include the data store itself.

[Next…  Extending the Persistence Domain to Send Events](../5/)