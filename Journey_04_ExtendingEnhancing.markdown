## Chapter 4
# Extending and Enhancing the Orders and Registrations Bounded Contexts 

*Exploring further*

# A Description of the Orders and Reservations Bounded Context 

The Orders and Reservations Bounded Context was described in some detail 
in the previous chapter. This chapter describes some changes that the 
team made in this bounded context during the second stage of their CQRS 
journey. 

The specific topics described in this chapter include:

* Improvements to the way that message correlation works with the 
  **ReservationProcessWorkflow**. This illustrates how aggregate
  instances within the bounded context can interact in a complex
  manner.
* Implementing a record locator to enable a registrant to retrieve an 
  order that was saved during a previous session. This illustrates
  adding some additional logic to the write-side that enables you to
  locate an aggregate instance without knowing its unique Id.
* Adding a countdown timer to the UI to enable a registrant to track how 
  much longer they have to complete an order. This illustrates
  enhancements to the write-side to support displaying rich information
  in the UI.
* Supporting orders for multiple seat types simultaneously. For example
  a registrant requests five seats for pre-conference event and eight
  seats for the full conference. This adds more complex business logic
  into the write-side.
* Supporting partially fulfilled orders where the only some of the
  seats that the registrant requests are available. This adds more
  complex business logic into the write-side.
* CQRS command validation using MVC. This illustrates how to make use
  of the model validation feature in MVC to validate your CQRS commands
  before you send them to the domain.


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

This chapter discusses the implementation of two user stories in 
addition to describing some changes and enhancements to the **Orders and 
Reservations** bounded context. 

### Implement a Login using a Record Locator

When a registrant creates an order for seats at a conference, the system 
generates a five character **order access code** and sends it to the 
registrant by email. The registrant can use her email address and the 
**order access code** on the conference web site to retrieve the order 
from the system at a later date. The registrant may wish retrieve the 
order to review it, or to complete the registration process by assigning 
attendees to seats. 

### Inform the Registrant How Much Time Remains to Complete an Order

When a registrant creates an order, the system reserves the seats 
requested by the registrant until the order is complete or the 
reservations expire. To complete an order, the registrant must submit 
her details, such as name and email address, and make a successful 
payment. 

To help the registrant, the system displays a count-down timer to inform 
the registrant how much time remains to complete the order before the 
seat reservations expire. 

### Enabling a Registrant to Create an Order that Includes Multiple Seat Types

When a registrant creates an order, the registrant may request different 
numbers of different seat types. For example, a registrant may request 
five seats for the full conference and three seats for the 
pre-conference workshop. 

### Handling Partially Fulfilled Orders

## Architecture 

What are the key architectural features? Server-side, UI, multi-tier, cloud, etc. 

# Patterns and Concepts 

* What are the primary patterns or approaches that have been adopted for this bounded context? (CQRS, CQRS/ES, CRUD, ...) 

* What were the motivations for adopting these patterns or approaches for this bounded context? 

* What trade-offs did we evaluate? 

* What alternatives did we consider? 

## Record Locators

The system uses **Access Codes** instead of passwords to avoid the 
overhead for the registrant of setting up an account with the system. 
Many registrants may use the system only once, so there is no need to 
create a permanent account with a user ID and a password. 

The system needs to be able to retrieve order information quickly based 
on the registrant's email address and access code. To provide a minimum 
level of security, the access codes that the system generates should not 
be predictable, and the order information that registrants can retrieve 
should not contain any sensitive information. 

## Querying the Read-side

The previous chapter focused on the write-side model and implementation, 
in this chapter we'll explore the read-side implementation in more 
detail. In particular you'll see how the team implemented the read model 
and the querying mechanism from the MVC controllers. 

In this initial exploration of the CQRS pattern, the team decided to use 
SQL views in the database as the underlying source of the data queried 
by the MVC controllers on the read-side. These views currently exist in 
the same database as the normalized tables that the write model uses. 

> **JanaPersona:** The team will split the database into two and explore
> options for pushing changes from the normalized write-side to the
> de-normalized read-side in a later stage of the journey.

### Storing De-normalized Views in a Database

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

In figure 1, a client such as an MVC controller action invokes a method 
on a **ViewRepository** class to request the data that it needs. The 
**ViewRepository** class in turn runs a query against the de-normalized 
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
  the underlying database. It is supported by multiple ORMs and
  minimizes the amount of code that you must write. 
* **Simplicity #2.** You only need to define a single repository and a 
  single **Query** method. 
* **Simplicity #3.** You don't need a separate query object.On the 
  read-side the queries should be simple because you have already 
  de-normalized your data to support the read-side clients. 
* **Simplicity #4.** You can make use of LINQ to provide support for 
  features such as filtering, paging, and sorting in the client. 
* **Testability.** You can use LINQ to Objects for mocking. 

> **MarkusPersona:** In the RI, using Entity Framework, we didn't need
> to write any code at all to expose the **IQueryable** instance. We
> also had just a single **ViewRepository** class. 

 

Possible objections to this approach include:

* It is not easy to replace the data store with a non-relational 
  database. However, you can choose to implement the write-model 
  differently in each bounded context using an approach that is 
  appropriate to that bounded context. 
* The client might abuse the **IQueryable** interface be performing 
  operations that can be done more efficiently as a part of the 
  de-normalization process. You should ensure that the de-normalized
  data fully meets the requirements of the clients. 
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
  LINQ, and the *IQueryable** interface behind the scenes to get the
  data from the data store. This is a choice that could be made on a
  method by method basis. 
* **Performance #1.** You can easily optimize the queries that the 
  **Find** and **Get** methods run. 
* **Performance #2.** The data access layer executes all queries. There 
  is no risk that the client MVC controller action tries to run complex 
  and inefficient LINQ queries against the data source. 
* **Testability.** It is easier to specify unit tests for the **Find** 
  and **Get** methods than to create suitable unit tests for the range
  of possible LINQ queries that a client could specify. 
* **Simplicity #1.** Dependencies are clearer for the client. For 
  example, the client receives an explicit **IOrderSummaryDAO** instance 
  rather than a generic **IViewRepository** instance. 
* **Simplicity #2.** For the majority of queries, there are only one or 
  two predefined ways to access the object. Different queries typically 
  return different projections. 

Possible objections to this approach include:

* Using the **IQueryable** interface makes it much easier to use grids 
that support features such as paging, filtering, and sorting in the UI. 

## CQRS Command Validation

When you implement the write-model, you should try to ensure that 
commands very rarely fail. This gives the best user experience, and 
makes it much easier to implement the asynchronous behavior in your 
application. 

One approach, adopted by the team, is to use the model validation 
features in ASP.NET MVC 3. 

You should be careful to distinguish between errors and business
failures. Examples of errors include: 

* A message not being delivered due to a failure in the messaging
  infrastructure.
* Data not being persisted due to a connectivity problem with the
  database.

In many cases, especially in the cloud, you can handle these errors by
 retrying the operation.

A business failure should have a predetermined business response. For
 example:

* If a seat cannot be reserved because there are no seats left, then the
  system should add the request to a wait-list.
* If a credit card payment fails, the user should be given the chance to
  try a different card, or to set up payment by invoice.

> **BharathPersona:** Your domain experts should help you to identify
> possible business failures and determine the way that you handle
> them. 

# Implementation Details 

This section describes some of the significant features of the 
implementation of the orders and reservations bounded context that are 
described in this chapter. You may find it useful to have a copy of the 
code so you can follow along. You can download a copy of the code from 
the repository on github: [mspnp/cqrs-journey-code][repourl]. 

## The Order Access Code Record Locator 

A registrant may need to retrieve an Order, either to view it, or to 
complete registering attendees to seats. This may happen in a different 
web session, so the registrant must supply some information to locate 
the previously saved order. 

The following code sample shows how the **Order** class generates an new 
five character order access code that is persisted as part of the 
**Order** instance. 

```Cs
public string AccessCode { get; set; }

protected Order()
{
	...
	this.AccessCode = HandleGenerator.Generate(5);
}
```

To retrieve an **Order** instance, a registrant must provide her email 
address and the order access code. The system will use these two items 
to locate the correct order. This logic is part of the read-side. 

The following code sample from the **OrderController** class in the web 
application shows how the MVC controller submits the query to the 
repository to discover the unique **OrderId** value. This **Find** 
action passes the **OrderId** value to a **Display** action that 
displays the order information to the registrant. 

```Cs
[HttpPost]
public ActionResult Find(string conferenceCode, string email, string accessCode)
{
	var repo = this.repositoryFactory();
	using (repo as IDisposable)
	{
		var order = repo.Query<OrderDTO>()
			.Where(o => o.RegistrantEmail == email && o.AccessCode == accessCode)
			.FirstOrDefault();

		if (order == null)
			return RedirectToAction("Find", new { conferenceCode = conferenceCode });

		return RedirectToAction("Display", new { conferenceCode = conferenceCode, orderId = order.OrderId });
	}
}
```

This illustrates two ways of querying the read-side. The first, as shown 
in the previous code sample, enables you to use a LINQ query against an 
**IQueryable** instance. The second, as used in the MVC controller's 
**Display** action, retrieves a single instance based on its unique Id. 
The following code sample shows the **Find** and **Query** methods in 
the **OrmViewRepository** class that the MVC controller uses. 

```Cs
public T Find<T>(Guid id) where T : class
{
	return this.Set<T>().Find(id);
}

public IQueryable<T> Query<T>() where T : class
{
	return this.Set<T>();
}
```

## The Count-down Timer

When a registrant creates and order and makes a seat reservation, those 
seats are reserved for a fixed period of time. The 
**ReservationWorkflow** instance, which forwards the reservation from 
the **SeatsAvailability** aggregate, passes the time that the 
reservation expires to the **Order** aggregate. The following code 
sample shows how the **Order** aggregate receives and stores the 
reservation expiry time. 

```Cs
public DateTime? BookingExpirationDate { get; private set; }

public void MarkAsBooked(DateTime bookingExpirationDate)
{
	if (this.State != States.Created)
		throw new InvalidOperationException();

	this.State = States.Booked;
	this.BookingExpirationDate = bookingExpirationDate;
}

```

The MVC **RegistrationController** class retrieves the order information 
on the read-side. The **OrderDTO** class includes the reservation expiry 
time that is then passed to the view using the **ViewBag** class, as 
shown in the following code sample. 

```Cs
[HttpGet]
public ActionResult SpecifyRegistrantDetails(string conferenceCode, Guid orderId)
{
	var repo = this.repositoryFactory();
	using (repo as IDisposable)
	{
		var orderDTO = repo.Find<OrderDTO>(orderId);
		var conferenceDTO = repo.Query<ConferenceDTO>()
			.Where(c => c.Code == conferenceCode)
			.FirstOrDefault();

		this.ViewBag.ConferenceName = conferenceDTO.Name;
		this.ViewBag.ConferenceCode = conferenceDTO.Code;
		this.ViewBag.ExpirationDateUTCMilliseconds = orderDTO.BookingExpirationDate.HasValue ? ((orderDTO.BookingExpirationDate.Value.Ticks - EpochTicks) / 10000L) : 0L;
		this.ViewBag.OrderId = orderId;

		return View(new AssignRegistrantDetails { OrderId = orderId });
	}
}
```
The MVC view then uses Javascript to display an animated count-down 
timer. 

## Using ASP.NET MVC 3 Validation for Commands

You should try to ensure that any commands that the MVC controllers in 
your application send to the write-model will succeed. As part of this, 
you can use the features in MVC to validate the commands both 
client-side and server-side before sending them to the write-model. 

> **MarkusPersona:** Client-side validation is primarily a convenience
> to the user that avoids the need to for round trips to the server to
> help the user complete a form correctly. You still need server-side
> validation to ensure that the data is validated before it is forwarded
> to the write-model.

The following code sample shows the **AssignRegistrantDetails** command 
class that uses **DataAnnotations** to specify the validation 
requirements; in this example, that the **FirstName**, **LastName**, and 
**Email** fields are not empty. 

```Cs
using System;
using System.ComponentModel.DataAnnotations;
using Common;

public class AssignRegistrantDetails : ICommand
{
	public AssignRegistrantDetails()
	{
		this.Id = Guid.NewGuid();
	}

	public Guid Id { get; private set; }

	public Guid OrderId { get; set; }

	[Required(AllowEmptyStrings = false)]
	public string FirstName { get; set; }

	[Required(AllowEmptyStrings = false)]
	public string LastName { get; set; }

	[Required(AllowEmptyStrings = false)]
	public string Email { get; set; }
}
```

The MVC view uses this command class as its model class. The following 
code sample from the **SpecifyRegistrantDetails.cshtml** file shows how 
the model is populated. 

```HTML
@model Registration.Commands.AssignRegistrantDetails

...

<div class="editor-label">@Html.LabelFor(model => model.FirstName)</div><div class="editor-field">@Html.EditorFor(model => model.FirstName)</div>
<div class="editor-label">@Html.LabelFor(model => model.LastName)</div><div class="editor-field">@Html.EditorFor(model => model.LastName)</div>
<div class="editor-label">@Html.LabelFor(model => model.Email)</div><div class="editor-field">@Html.EditorFor(model => model.Email)</div>

```

Client-side validation based on the **DataAnnotations** attributes is 
configured in the **Web.config** file as shown in the following snippet. 

```XML
<appSettings>
    ...
    <add key="ClientValidationEnabled" value="true" />
    <add key="UnobtrusiveJavaScriptEnabled" value="true" />
</appSettings>
```

The server-side validation occurs in the controller before the command 
is sent. The following code sample from the **RegistrationController** 
class shows how the controller uses the **IsValid** property to validate 
the command. Remember that this example uses an instance of the command 
as the model. 

```Cs
[HttpPost]
public ActionResult SpecifyRegistrantDetails(string conferenceCode, Guid orderId, AssignRegistrantDetails command)
{
    if (!ModelState.IsValid)
    {
        return SpecifyRegistrantDetails(conferenceCode, orderId);
    }

    this.commandBus.Send(command);

    return RedirectToAction("SpecifyPaymentDetails", new { conferenceCode = conferenceCode, orderId = orderId });
}
```

For more information, see [Models and Validation in ASP.NET 
MVC][modelvalidation] on MSDN. 

## Refactoring the SeatsAvailability Aggregates

In the first stage of our CQRS, the domain included a 
**ConferenceSeatsAvailabilty** aggregate root class that modelled the 
number of seats remaining for a conference. In this stage of the 
journey, the team replaced the **ConferenceSeatsAvailabilty** aggregate 
with a **SeatsAvailability** aggregate to reflect the fact that there 
may be multiple seat types available at a particular conference; for 
example, full conference seats, pre-conference workshop seats, and 
cocktail party seats. Figure 2 shows the new **SeatsAvailability** 
aggregate and its constituent classes. 

![Figure 2][fig2]

**The **SeatsAvailability** and its associated commands and events.

This aggregate now models the following facts:

* There may be multiple seat types at a conference.
* There may be different numbers of seats available for each seat type.

The domain now includes a **SeatQuantity** value type that you can use 
to represent a quantity of a particular seat type. 

Previously, the aggregate raised either a **ReservationAccepted** or 
**ReservationRejected** event depending on whether there were sufficient 
seats. Now the aggregate raises a **SeatsReserved** event that reports 
how many seats of a particular type could be reserved. This means that 
the number of seats reserved may not match the number of seats 
requested; this information is passed back to the UI for the registrant 
to make a decision on how to proceed with the registration. 

### The AddSeats Method

You may have noticed in Figure 2 that the **SeatsAvailability** 
aggregate includes an **AddSeats** method with no corresponding command. 
The **AddSeats** method adjusts the total number of available seats of a 
given type. The conference owner is responsible for making any such 
adjustments, and does this in the Conference Management bounded context. 
The Conference Management bounded context raises an event whenever the 
total number of available seats changes, the **SeatsAvailability** class 
then handles the event when its handler invokes the **AddSeats** method. 

# Testing 

Describe any special considerations that relate to testing for this bounded context.  


[r_chapter4]:     Reference_04_DeepDive.markdown

[fig1]:           images/Journey_04_ViewRepository.png?raw=true

[modelvalidation]: http://msdn.microsoft.com/en-us/library/dd410405(VS.98).aspx