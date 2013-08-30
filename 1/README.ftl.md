 

## Modelling the Core Domain

TODO Zoomed in on the core components of the system in the Life Preserver.

Currently the core, application internal domain of the Yummy Noodle Bar is made up of the following components:

* **Orders**

    The collection of all orders currently in the system, regardless of status. In the terminology of [Domain Driven Design](http://en.wikipedia.org/wiki/Domain-driven_design), Orders is an Aggregate Root that ensures consistency across all of the Orders in the system.

* **Order**

    An individual order in the system that has an associated status and status history for tracking purposes.

* **OrderStatus**

    The current allocated status to an order.

* **OrderStatusHistory**

    Associated with an order, this is an ordered collection of the previous status' that the order has transitioned through.

* **PaymentDetails**
* **PaymentStatus**
* **Menu**
* **MenuItem**
* **MenuItemAvailability**

Focussing primarily on Orders, these can be acted upon by a number of events:

* **OrderCreatedEvent**

    Creates a new order for a number of menu-items.

* **OrderUpdatedEvent**

    Updates an existing Order with some additional information, possibly payment information.

* **OrderDeletedEvent**

    Deletes an existing order if it is not being cooked.

* **RequestAllCurrentOrdersEvent**

    Requests the full list of all current orders be returned.

## Modelling the Persistence Domain

For the first version of the Yummy Noodle Bar persistence services, the ability to created, update and remove Orders, MenuItems and OrderStatuses is the focus.

It can be tempting to simply map the core Order domain to the data stores and work from there, but that would ignore the boundary between the Core and the Persistence domain.

(TODO highlight this boundary on a focus on the Life Preserver).

The data model of your persisted data will need to change at a rate that is manageable and technically feasible given the data store implementation, and the core will need to evolve at whatever rate the Yummy Noodle bar system needs to internally evolve at. So there is potentially friction between the two domains as they may need to evolve at different rates.

To manage this friction you need to create concepts and components in the Persistence Service domain that are unique to, and can evolve at the rate needed by, the Persistence domain itself. This may result in similar types of components but since their purpose will be very different the similarities are superficial.

## Understanding differing data models and their implementations

Modern data stores adopt one or more Data Models, although typically only one per technology. The way that you can expect to interact with a data store, based on the features and limits of its data model, will be different and so different capabilities and data access patterns will be needed.

Some of the most common Data Models currently in use are:

**Relational**

The relational model of data storage is characterised by the concepts of Tables, with structured Columns made up of Rows.  This is is very analogous to a spreadsheet, where a table is like a worksheet, a row represents one record, and a column contains one specific piece of information. They are accessed by a dialect of the Structured Query Language (SQL). 

What makes this type of data store relational is the way that Columns may define references to Columns in other tables, known as relations. These relations are usually strongly enforced by the database.

Relational Databases are highly structured, have an explicit Schema and almost invariably first class support for transactions.

You will see the H2 Database in use in this tutorial, supported by [Spring Data JPA](http://www.springsource.org/spring-data/jpa)
    
**Document**

A Data Store following the Document Data Model will have a far looser schema than a relational database, if it has one at all. Documents are structured, rich data structures that can contain nested documents, lists, maps and other constructs internally, all within a single document. Some elements of the documents can be optional, giving huge flexibility to the system designer.

Queries can be written against any value in the document structure, leading to very rich querying capabilities and document structures.  I distinct difference with relational databases is that the logical collections of documents do not have to conform to a common schema. Documents aren't as heavily defined by relationships between columns, but instead more focused on the content of the documents.

You will see MongoDB in use in this tutorial, supported by [Spring Data MongoDB](http://www.springsource.org/spring-data/mongodb)

**Key/Value**

Key/Value is one of the simplest forms of Data Model.  It is analogous to a Java Map, known in other languages as either a hash table or an associative array. The data entries involve just a key with an associated value. They tend to be very fast and easily distributable. This is sometimes viewed as a caching data solution. Because many critical, high performance systems depend on such simple constructs, Key/Value data stores have risen quickly in popularity.

Some implementations offer a querying capability against the value data.

No implementation of a simple key/value store is used in this tutorial. Redis is supported by [Spring Data Redis](http://www.springsource.org/spring-data/redis)

**Data Grid**

Data Grid is less well defined than the other models, but mostly commonly means a Key/Value store that has advanced replication and server side data processing built in. This is sometimes viewed as a caching data solution.

GemFire is such a store, and allows accessing data through either a Map interface, a rich query language, or writing code that executes on the GemFire cluster in a distributed fashion.

It is supported by [Spring Data GemFire](http://www.springsource.org/spring-gemfire)

[Next… Storing Menu Data Using MongoDB](../2/)




