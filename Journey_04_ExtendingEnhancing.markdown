## Chapter 4
# Extending and Enhancing the Orders and Registrations Bounded Contexts 

*Exploring further*

# A Description of the Orders and Reservations Bounded Context 

The Orders and Reservations Bounded Context was discribed in some detail 
in the previous chapter. This chapter describes some changes that the 
team made in this bounded context during the second stage of their CQRS 
journey. 

The specific topics described in this chapter include:

* Improvements to the way that message correlation works with the 
  **ReservationProcessWorkflow**. 
* Implementing a record locator to enable a registrant to retrieve an 
  order that was saved during a previous session. 
* Adding a countdown timer to the UI to enable a registrant to track how 
  much longer tey have to complete an order. 


## Working Definitions for this Chapter 

The following definitions are used for the remainder of this chapter. 
For more detail, and possible alternative definitions see [A CQRS/ES 
Deep Dive][r_chapter4] in the Reference Guide. 

### Command

A command is a request for the system to perform a task or an action. 
Commands are imperatives, for example **MakeRegistration**. In this 
bounded context, commands originate from the UI as a result either of 
the user initiating a request or from a saga when the saga is directing 
an aggregate to perform an action. 

Commands are processed once, and once only by a single recipient. A 
command bus transports commands that command handlers then dispatch to 
aggregates. Sending a command is an asynchronous operation with no 
return value. 

### Event

An event describes something that has happened in the system, typically 
as a result of a command. Aggregates in the domain model raise events. 

Multiple subscribers can handle a specific event. Aggregates publish 
events to an event bus; handlers register for specific types of event on 
the event bus and then deliver the event to the subscriber. In this 
bounded context, the only subscriber is a saga. 

### Workflow

In this bounded context, a workflow is a class that coordinates the 
behavior of the aggregates in the domain. A workflow subscribes to the 
events that the aggregates raise, and then follow a simple set of rules 
to determine which command or commands to send. The workflow does not 
contain any business logic, simply logic to determine the next command 
to send. The workflow is implemented as a state machine, so when the 
workflow responds to an event, it can change its internal state in 
addition to sending a new command. 

The workflow in this bounded context can receive commands as well as 
subscribe to events. 

## User Stories 

This chapter discusses the implementation of two user stories in addition to describing some changes and enhancements to the **Orders and Reservations** bounded context.

### Implement a Login using a Record Locator

When a registrant creates an order for seats at a conference, the system generates a five character **order access code** and sends it to the registrant by email. The registrant can use her email address and the **order access code** on the conference web site to retrieve the order from the system at a later date. The registrant may wish retrieve the order to review it, or to complete the registration process by assigning attendees to seats.

### Inform the Registrant How Much Time Remains to Complete an Order.

When a registrant creates an order, the system reserves the seats requested by the registrant until the order is complete or the reservations expire. To complete an order, the registrant must submit her details, such as name and email address, and make a successful payment.

To help the registrant, the system displays a count-down timer to inform the registrant how much time remains to complete the order before the reservations expire.

## Architecture 

What are the key architectural features? Server-side, UI, multi-tier, cloud, etc. 

# Patterns and Concepts 

* What are the primary patterns or approaches that have been adopted for this bounded context? (CQRS, CQRS/ES, CRUD, ...) 

* What were the motivations for adopting these patterns or approaches for this bounded context? 

* What trade-offs did we evaluate? 

* What alternatives did we consider? 

The previous chapter focused on the write-side model and implementation, 
in this chapter we'll explore the read-side implementation in more 
detail. In particular you'll see how the team implemented the read model 
and the querying mechanism from the MVC controllers. 

In this initial exploration of the CQRS pattern, the team decided to use 
SQL views in the database as the underlying source of the data queried 
by the MVC controllers on the read-side. These views currently exist in 
the same database as the normalized tables that the write model uses. 

> **JanaPersona:** The team will split the database into two and explore
  options for pushing changes from the normalized write-side to the
  de-normalized read-side in a later stage of the journey.

## Storing De-normalized Views in a Database

One common option for storing the read-side data is to use a set of 
relational database tables to hold the de-normalized views. The 
read-side should be optimized for fast reads, so there is typically no 
benefit in storing normalized data because this will require complex 
queries to construct the data for the client. This implies that goals 
for the read-side should be to keep the queries as simple as possible, 
and to structure the tables in the database in such a way that they can 
be read quickly and efficiently. 

An important area for consideration is the interface whereby a client 
such as an MVC controller action submits a query to the read-side model. 

![Figure 1][fig1]

**The Read-side storing data in a relational database**

In figure 4, a client such as an MVC controller action invokes a method 
on a **ViewRepository** class to request the data that it needs. The 
**ViewRepository** class in turn runs a query against the denormalized 
data in the database. 

One approach to consider for the **ViewRepository** class is to have it 
return an **IQueryable** instance that enables the client to use LINQ to 
specify its query. It is very easy to return an **IQueryable** instance 
from many ORMs such as Entity Framework or NHibernate. The following 
code snippet illustrates how the client can submit such queries. 

```Cs
var ordersummaryDTO = repository.Query<OrderSummaryDTO>().Where(LINQ query to retrieve order summary);
var orderdetailsDTO = repository.Query<OrderDetailsDTO>().Where(LINQ query to retrieve order details);
```

This approach has a number of advantages:

* **Simplicity #1.** This approach uses a thin abstraction layer over 
the underlying database. It is supported by multiple ORMs and minimizes 
the amount of code that you must write. 
* **Simplicity #2.** You only need to define a single repository and a 
single **Query** method. 
* **Simplicity #3.** You don't need a separate query object.On the 
read-side the queries should be simple because you have already 
de-normalized your data to support the read-side clients. 
* **Simplicity #4.** You can make use of LINQ to provide support for 
features such as filtering, paging, and sorting in the client. 
* **Testability.** You can use LINQ to Objects for mocking. 

> **MarkusPersona** In the RI, using Entity Framework, we didn't need to 
write any code at all to expose the **IQueryable** instance. We also had 
just a single **ViewRepository** class. 

Possible objections to this approach include:

* It is not easy to replace the data store with a non-relational 
database. However, you can choose to implement the write-model 
differently in each bounded context using an approach that is 
appropriate to that bounded context. 
* The client might abuse the **IQueryable** interface be performing 
operations that can be done more efficiently as a part of the 
de-normalization process. You should ensure that the de-normalized data 
fully meets the requirements of the clients. 
* Using the **IQueryable** interface hides the queries away. However, 
since you can de-normalize the data on the write-side, the queries 
against the relational database tables are unlikely to be complex. 

An alternative approach is to have the **ViewRepository** expose custom 
**Find** and **Get** methods as shown in the following code snippets. 

```Cs
var ordersummaryDTO = dao.FindAllSummarizedOrders(userId);
var orderdetailsDTO = dao.GetOrderDetails(orderId);
```

You could also choose to use different DAO classes. This would make it 
easier to access different data sources. 

```Cs
var ordersummaryDTO = OrderSummaryDAO.FindAll(userId);
var orderdetailsDTO = OrderDetailsDAO.Get(orderId);
```

This approach has a number of advantages:

* **Flexibility #1.** The **Get** and **Find** methods hide details such 
as the partitioning of the data store and the data access methods such 
as an ORM or executing SQL code explicitly. This makes it easier to 
change these choices in the future. 
* **Flexibility #2.** The **Get** and **Find** methods could use an ORM, 
LINQ, and the *IQueryable** interface behind the scenes to get the data 
from the data store. This is a choice that could be made on a method by 
method basis. 
* **Performance #1.** You can easily optimize the queries that the 
**Find** and **Get** methods run. 
* **Performance #2.** The data access layer executes all queries. There 
is no risk that the client MVC controller action tries to run complex 
and inefficient LINQ queries against the data source. 
* **Testability.** It is easier to specify unit tests for the **Find** 
and **Get** methods than to create suitable unit tests for the range of 
possible LINQ queries that a client could specify. 
* **Simplicity #1.** Dependencies are clearer for the client. For 
example, the client receives an explicit **IOrderSummaryDAO** instance 
rather than a generic **IViewRepository** instance. 
* **Simplicity #2.** For the majority of queries, there are only one or 
two predefined ways to access the object. Different queries typically 
return different projections. 

Possible objections to this approach include:

* Using the **IQueryable** interface makes it much easier to use grids 
that support features such as paging, filtering, and sorting in the UI. 

# Implementation Details 

Describe significant features of the implementation with references to the code. Highlight any specific technologies (including any relevant trade-offs and alternatives). 

Provide significantly more detail for those BCs that use CQRS/ES. Significantly less detail for more "traditional" implementations such as CRUD. 

# Testing 

Describe any special considerations that relate to testing for this bounded context.  


[fig4]:           images/Journey_04_ViewRepository.png?raw=true