## Chapter 5
# Preparing for the V1 Release  

*Adding functionality and refactoring in preparation for the V1 release.*

# A Description of the Contoso Conference Management V1 Release

This chapter describes the changes made by the team to prepare for the 
first production release of the Contoso Conference Management System. 
This work includes some refactoring and additions to the Orders and 
Registrations bounded context that was introduced in the previous two 
chapters as well as a new Conference Management bounded context. 

One of the key refactorings undertaken by the team during this phase of 
the journey was to introduce event sourcing into the Orders and 
Registrations bounded context. 

One of the anticipated benefits from implementing the CQRS pattern is 
that it will help to manage change in a complex system. Having a V1 
release during the CQRS journey will help the team to evaluate how the 
CQRS pattern and event sourcing deliver these benefits when they move 
forward from the V1 release to the next production release of the 
system. The following chapters will describe what happens after the V1 
release. 

This chapter also describes the Metro-style UI that the team added to 
the public web-site during this phase and includes a discussion of 
task-based UIs. 

## Working Definitions for this Chapter 

Outline any working definitions that were adopted for this chapter. 

### Access Code

When a Business Customer creates a new Conference, the system generates 
a five character Access Code and sends it by email to the Business 
Customer. The Business Customer can use his email address and the Access 
Code on the Conference Management Web Site to retrieve the conference 
details from the system at a later date. The system uses access codes 
instead of passwords to avoid the overhead for the Business Customer of 
setting up an account with the system.

### Event Sourcing

Event Sourcing is a way of persisting and reloading the state of 
aggregates within the system. Whenever the the state of an aggregate 
changes, the aggregate raises an event detailing the state change. The 
system then saves this event in an event store. The system can recreate 
the state of an aggregate by replaying all of the previously saved 
events associated with that aggregate instance. The event store becomes 
the book of record for the data stored by the system. 

In addition, you can use event sourcing as a source of audit data, as a 
way to query historic state, and to replay events for debugging and 
problem analysis. 

## User Stories 

The team implemented the user stories listed below during this phase of 
the project. 

### Conference Management User Stories

The **Business Customer** represents the organization that is using the 
conference management system to run its conference. 

A **Seat** represents a space at a conference or access to a specific 
session at the conference such as a cocktail party, a tutorial, or a 
workshop. 

A business customer can create new conferences and manage them. After a 
business customer creates a new conference, he can access the details of 
the conference by using his email address and conference locator access 
code. The system generates the access code when the business customer 
creates the conference. 

The business customer can specify the following information about a 
conference: 

* The name, description, and slug (part of the URL used to access the
  conference).
* The start and end dates of the conference.
* The different types and quotas of seats available at the conference.

Additionally, the business customer can control the visibility of the 
conference on the public web-site by either publishing or un-publishing 
the conference. 

The business customer can use the conference management web-site to view 
a list of attendees. 

### Ordering and Registration User Stories

After a registrant has purchased seats at a conference, she can assign 
attendees to those seats. The system stores the name and contact details 
for each attendee. 

## Architecture 

What are the key architectural features? Server-side, UI, multi-tier, cloud, etc.

## Conference Management Bounded Context

The Conference Management bounded context is a simple two-tier, 
CRUD-style web application. It is implemented using MVC 4 and Entity 
Framework. 

**JanaPersona:** The team implemented this bounded context after it 
implemented the public Conference Management web-site that uses MVC 3. 

# Patterns and Concepts 

* What are the primary patterns or approaches that have been adopted for this bounded context? (CQRS, CQRS/ES, CRUD, ...) 

* What were the motivations for adopting these patterns or approaches for this bounded context? 

* What trade-offs did we evaluate? 

* What alternatives did we consider? 

## Event Sourcing

The team at Contoso originally implemented the Orders and Reservations 
bounded context without using event sourcing. However, during the 
implementation it became clear that using event sourcing would help to 
simplify this bounded context. 

In the previous chapter [Extending and Enhancing the Orders and 
Registrations Bounded Contexts][j_chapter4] the team found that they 
needed to use events to push changes from the write-side to the 
read-side. On the read-side the **OrderViewModelGenerator** class 
subscribed to the events published by the **Order** aggregate, and used 
those events to update the views in the database that were queried by 
the read-model. 

This was already half-way to an event sourcing implementation, so it 
made sense to use a single persistence mechanism based on events for the 
whole bounded context. 

The event sourcing infrastructure is reusable in other bounded contexts, 
and the implementation of the Orders and Registrations becomes simpler. 

> **PoePersona:** As a practical problem, the team had limited time
> before the V1 release to implement a production quality event store.
> They created a simple, basic event store based on SQL Server as an
> interim solution. However, they will face the problem in the future
> of migrating from one event store to another.

## Task-based UI

The design of UIs has improved greatly over the last decade: 
applications are easier to use, more intuitive, and simpler to navigate 
than they were before. Examples of guidelines for UI designers are the 
[Microsoft Inductive User Interface Guidelines][inductiveui] and the [UX 
guidelines for Metro style apps][metroux]. 

Another factor that affects the design and usability of the UI is how 
the UI communicates with the rest of the application. If the application 
is based on a CRUD-style architecture, this can leak through to the UI. 
If the developers focus on CRUD-style operations, this can result in a 
UI as shown in the first screen design in Figure 1. 

![Figure 1][fig1]

**Example UIs for conference registration**

On the first screen, the labels on the buttons reflect the underlying 
CRUD operations that the system will perform when the user clicks the 
**Submit** button. The first screen also requires the user to apply some 
deductive knowledge about how the screen and the application function. 
For example, the function of the **Add** button is not immediately 
apparent. 

A typical implementation behind the first screen will use a data 
transfer object (DTO) to exchange data between the back-end and the UI. 
The UI will request data from the back-end that will arrive encapsulated 
in a DTO, it will modify the data in the DTO, and then return the DTO 
to the back-end. The back-end will use the DTO to figure out what CRUD 
operations it must perform on the underlying datastore. 

The second screen is more explicit about what is happening in terms of 
the business process: the user is selecting quantities of seat types as 
a part of the conference registration task. Thinking about the UI in 
terms of the task that the user is performing makes it easier to relate 
the UI to the write-model in your implementation of the CQRS pattern. 
The UI can send commands to the write-side, and those commands are a 
part of the domain model on the write-side. In a bounded context that 
implements the CQRS pattern, the UI typically queries the read-side and 
receives a DTO, and sends commands to the write-side. 

![Figure 2][fig2]

**Task-based UI flow**

Figure 2 shows a sequence of pages that enable the registrant to 
complete the "purchase seats at a conference" task. On the first page, 
the registrant selects the type and quantity of seats. On the second 
page, the registrant can review the seats she has reserved, enter her 
contact details, and complete the necessary payment details. The system 
then redirects the registrant to a payment provider, and if the payment 
completes successfully, the system displays the third page. The third 
page shows a summary of the order and provides a link to pages where the 
registrant can start additional tasks. 

The sequence shown in Figure 2 is deliberately simplified in order to 
highlight the roles of the commands and queries in a task-based UI. For 
example, the real flow includes pages that the system will display based 
on the payment type selected by the registrant, and error pages that the 
system displays if the payment fails. 

> **BharathPersona:** You don't always need to use task-based UIs. In
> some scenarios, simple CRUD-style UIs work well. You must evaluate
> whether benefits of task-based UIs outweigh the additional 
> implementation effort of a task-based UI. Very often, the bounded
> contexts where you choose to implement the CQRS pattern, are also the
> bounded contexts that benefit from task-based UIs because of the 
> more complex business logic and more complex user interactions. 

## CRUD

You should not use the CQRS pattern as part of your top-level 
architecture: you should implement the pattern only in those bounded 
contexts where it brings clear benefits. In the Contoso Conference 
Management System, the conference management bounded context is a 
relatively simple, stable, and low volume element of the overall system. 
Therefore the team decided that they would implement this bounded 
context using a traditional two-tier, CRUD-style architecture. 

## Integration between Bounded Contexts

The Conference Management bounded context needs to integrate with the 
Orders and Registrations bounded context. For example, if the business 
customer changes the quota for a seat type in the Conference Management 
bounded context, this change needs to be propagated to the Orders and 
Registrations bounded context. Also, if a registrant adds a new attendee 
to a conference, the business customer must be able to view details of 
the attendee in the list in the Conference Management web-site. 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Add details of integration options: db-centric, events, read-model.
	Also integration for changes in Seat quotas, new conferences and seat types.
  </span

# Implementation Details 

Describe significant features of the implementation with references to the code. Highlight any specific technologies (including any relevant trade-offs and alternatives). 

Provide significantly more detail for those BCs that use CQRS/ES. Significantly less detail for more "traditional" implementations such as CRUD. 

## The Conference Management Bounded Context

The Conference Management Bounded Context that enables a Business Customer to define and  manage conferences is implemented using a simple two-tier, CRUD-style application using MVC 4.

In the Visual Studio solution, the **Conference** project contains the model code, and the **Conference.Web** project contains the MVC views and controllers.

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Add details of intregration with other bounded contexts.
  </span>
</div> 

## Event Sourcing

The intial implementation of the event sourcing infrastructure is 
extremely basic: the team intends to replace it with a production 
quality event store in the near future. This section describes the 
initial, basic, implementation and lists the various ways it must be 
improved. 

The core elements of this basic event sourcing solution are:

* Whenever the state of an aggregate instance changes, the instance
  raises an event that fully describes the state change.
* The system persists these events in an event store. This solution uses
  a SQL database to store the events.
* An aggregate can rebuild its state by replaying its past stream of
  events.
* Other aggregates and workflows (possibly in different bounded
  contexts) can subscribe to these events.
  
### Raising Events when the State of an Aggregate Changes

The following two methods from the **Order** aggregate are examples of 
methods that the **OrderCommandHandler** class invokes when it receives 
a command for the order. Neither of these methods update the state of 
the **Order** aggregate, instead they raise an event that will be 
handled by the **Order** aggregate. In the **MarkAsReserved** method, 
there is some minimal logic to determine which of two events to raise. 


```Cs
public void MarkAsReserved(DateTime expirationDate, IEnumerable<SeatQuantity> reservedSeats)
{
    if (this.isConfirmed)
        throw new InvalidOperationException("Cannot modify a confirmed order.");

    var reserved = reservedSeats.ToList();

    // Is there an order item which didn't get an exact reservation?
    if (this.seats.Any(item => !reserved.Any(seat => seat.SeatType == item.SeatType && seat.Quantity == item.Quantity)))
    {
        this.Update(new OrderPartiallyReserved(this.id, this.Version + 1, expirationDate, reserved));
    }
    else
    {
        this.Update(new OrderReservationCompleted(this.id, this.Version + 1, expirationDate, reserved));
    }
}

public void ConfirmPayment()
{
    this.Update(new OrderPaymentConfirmed(this.id, this.Version + 1));
}
```

The **Update** method and the **Version** property are defined in the 
abstract bas class of the **Order** class. The following code sample 
shows this method and property in the **EventSourcedAggregateRoot** 
class. 


```Cs
public int Version { get { return this.version; } }

protected void Update(IDomainEvent e)
{
    this.handlers[e.GetType()].Invoke(e);
    Debug.Assert(e.Version == this.version + 1);
    this.version = e.Version;
    this.pendingEvents.Add(e);
}
```

The **Update** method determines which event handler in the aggregate it 
should invoke to handle the event type. 


> **MarkusPersona:** The version of the aggregate is incremented every
> time its state is updated.

The following code sample shows the event handler methods in the 
**Order** class that are invoked when the command methods shown above 
are called. 


```Cs
private void OnOrderPartiallyReserved(OrderPartiallyReserved e)
{
    this.seats = e.Seats.ToList();
}

private void OnOrderReservationCompleted(OrderReservationCompleted e)
{
    this.seats = e.Seats.ToList();
}

private void OnOrderExpired(OrderExpired e)
{
}

private void OnOrderPaymentConfirmed(OrderPaymentConfirmed e)
{
    this.isConfirmed = true;
}
```

These methods update the state of the aggregate.

An aggregate must be able to handle both events from other aggregates 
and events that it raises itself. The protected constructor in the 
**Order** class lists all the events that the **Order** aggregate can 
handle. 

```Cs
protected Order()
{
    base.Handles<OrderPlaced>(this.OnOrderPlaced);
    base.Handles<OrderUpdated>(this.OnOrderUpdated);
    base.Handles<OrderPartiallyReserved>(this.OnOrderPartiallyReserved);
    base.Handles<OrderReservationCompleted>(this.OnOrderReservationCompleted);
    base.Handles<OrderExpired>(this.OnOrderExpired);
    base.Handles<OrderPaymentConfirmed>(this.OnOrderPaymentConfirmed);
    base.Handles<OrderRegistrantAssigned>(this.OnOrderRegistrantAssigned);
}
```

### Persisting Events to the Event Store

When the aggregate processes an event in the **Update** method in the 
**EventSourcedAggregateRoot** class, it adds the event to a private list 
of pending events. This list is exposed as a public, **IEnumerable** 
property called **Events**. 

The following code sample from the **OrderCommandHandler** class shows 
how the handler invokes a method in the **Order** class to handle a 
command, and then uses a repository to persist the current state of the 
**Order** aggregate. 


```Cs
public void Handle(MarkSeatsAsReserved command)
{
    var order = repository.Find(command.OrderId);

    if (order != null)
    {
        order.MarkAsReserved(command.Expiration, command.Seats);
        repository.Save(order);
    }
}
```

The following code sample shows the initial, simple implementation of 
the **Save** method in the **SqlEventRepository** class. 


```Cs
public void Save(T aggregateRoot)
{
    // TODO: guarantee that only incremental versions of the event are stored
    var events = aggregateRoot.Events.ToArray();
    using (var context = this.contextFactory.Invoke())
    {
        foreach (var e in events)
        {
            using (var stream = new MemoryStream())
            {
                this.serializer.Serialize(stream, e);
                var serialized = new Event { AggregateId = e.SourceId, Version = e.Version, Payload = stream.ToArray() };
                context.Set<Event>().Add(serialized);
            }
        }

        context.SaveChanges();
    }

    // TODO: guarantee delivery or roll back, or have a way to resume after a system crash
    this.eventBus.Publish(events);
}
```

### Replaying Events to Re-build State

When a handler class loads an aggregate instance from storage, it loads the state of the instance by replaying the stored event stream.

The following code sample from the **OrderCommandHandler** class shows how this process is initiated by calling the **Find** method in the repository.

```Cs
public void Handle(MarkSeatsAsReserved command)
{
    var order = repository.Find(command.OrderId);

    ...
}
```

The following code sample shows how the **SqlEventRepository** class 
loads the event stream associated with the aggregate. 


```Cs
public T Find(Guid id)
{
    List<Event> all;
    using (var context = this.contextFactory.Invoke())
    {
        all = context.Set<Event>().Where(x => x.AggregateId == id).OrderBy(x => x.Version).ToList();
    }

    if (all.Count > 0)
    {
        var deserialized = all.Select(x => this.serializer.Deserialize(new MemoryStream(x.Payload))).Cast<IDomainEvent>().ToList();
        return (T)Activator.CreateInstance(typeof(T), deserialized);
    }

    return null;
}
```

The following code sample shows the constructor in the **Order** class 
that rebuilds the state of an order from its event stream when it is 
invoked by the **CreateInstance** method in the previous code sample. 


```Cs
public Order(IEnumerable<IDomainEvent> history) : this()
{
    this.Rehydrate(history);
}
```

The **Rehydrate** method is defined in the **EventSourcedAggregateRoot** 
class as shown in the following code sample. 

```Cs
protected void Rehydrate(IEnumerable<IDomainEvent> pastEvents)
{
    foreach (var e in pastEvents)
    {
        this.handlers[e.GetType()].Invoke(e);
        this.version = e.Version;
    }
}
```

For each stored event in the history, it determines the appropriate 
handler method to invoke in the **Order** class.

### Issues with the Simple Event Store Implementation

The simple implementation of event sourcing and an event store outlined 
in the previous sections has a number of short-comings. The following 
list identifies some of these short-comings that should be overcome in a 
production quality implmentation. 

1. There is no guarantee in the **Save** method in the
   **SqlEventRepository** class that the event is persisted to storage
   and published to the messaging infrastructure. A failure could result
   in an event being saved to storage but not being published.
2. There is no chack that when the system persists an event, that it is
   a later event than the previous one. Potentially, events could be
   stored out of sequence.
3. There are no optimizations in place for aggregate instances that have
   a large number of events in their event stream. This could result in
   performance problems when replaying events.

# Running the Applications

You can run the Contoso Conference Management System in two modes: 
either deployed to Windows Azure, or running locally. 

If you deploy the V1 Release of the Contoso Conference Management System 
to Windows Azure, then the application uses Windows Azure Service Bus to 
provide its messaging infrastructure using brokered messages. If you run 
the application locally, then it runs using the Windows Azure Compute 
Emulator and uses the **MemoryEventBus** and **MemoryCommandBus** 
classes in the Common project to provide messaging services for events.

## Running the Contoso Conference Management System Locally

The following procedure describes how to configure the Contoso 
Conference Management System to run locally without using Windows Azure. 
Running the application in this way is useful if you want to quickly 
explore how the application works without configuring Windows Azure 
accounts. 

**To Run the Contoso Conference Management System Locally**

1. In Visual Studio, open the **Conference** solution.
2. On the **Build** menu, click **Configuration Manager**.
3. In the **Active solution configuration** drop-down list, select
   **DebugLocal**, then click **Close**.
4. You can now build and run the solution locally.

## Running the Contoso Conference Management System on Windows Azure

The following procedure describes how to configure the Contoso 
Conference Management System to run on Windows Azure. 
Running the application in this way is more realistic, but you must
complete some additional configuration steps. 

**To Run the Contoso Conference Management System on Windows Azure**

1. In Visual Studio, open the **Conference** solution.
2. On the **Build** menu, click **Configuration Manager**.
3. In the **Active solution configuration** drop-down list, select
   **Debug**, then click **Close**.
4. In **Solution Explorer**, in the **Azure** folder, right-click the
   **Settings.Template.xml" file, and then click **Rename**.
5. Rename the the **Settings.Template.xml** file to **Settings.xml**.
6. In **Solution Explorer**, double-click the **Settings.xml** file to
   open it in the editor.
7. Follow the instructions in the file to enter your Windows Azure
   account details.
8. On the **File** menu, click **Save Settings.xml**.
4. You can now build the solution and and deploy it to Windows Azure.

> **Note:** Other projects in the solution have links to the
> **Settings.xml** file in order to access the Windows Azure account
>  information.

## Conditional Compilation Notes
 
The following code sample from the **Global.asax.cs** file in the web 
projects shows how the project uses conditional compilation to select 
between the implementations.

```Cs
#if LOCAL
            container.RegisterType<ICommandBus, MemoryCommandBus>(new ContainerControlledLifetimeManager());
            container.RegisterType<ICommandHandlerRegistry, MemoryCommandBus>(new ContainerControlledLifetimeManager(), new InjectionFactory(c => new MemoryCommandBus()));
            container.RegisterType<IEventBus, MemoryEventBus>(new ContainerControlledLifetimeManager());
            container.RegisterType<IEventHandlerRegistry, MemoryEventBus>(new ContainerControlledLifetimeManager(), new InjectionFactory(c => new MemoryEventBus()));
#else
            var serializer = new JsonSerializerAdapter(JsonSerializer.Create(new JsonSerializerSettings
            {
                // Allows deserializing to the actual runtime type
                TypeNameHandling = TypeNameHandling.Objects,
                // In a version resilient way
                TypeNameAssemblyFormat = System.Runtime.Serialization.Formatters.FormatterAssemblyStyle.Simple
            }));

            var settings = MessagingSettings.Read(HttpContext.Current.Server.MapPath("bin\\Settings.xml"));
            var commandBus = new CommandBus(new TopicSender(settings, "conference/commands"), new MetadataProvider(), serializer);
            var eventBus = new EventBus(new TopicSender(settings, "conference/events"), new MetadataProvider(), serializer);

            var commandProcessor = new CommandProcessor(new SubscriptionReceiver(settings, "conference/commands", "all"), serializer);
            var eventProcessor = new EventProcessor(new SubscriptionReceiver(settings, "conference/events", "all"), serializer);

            container.RegisterInstance<ICommandBus>(commandBus);
            container.RegisterInstance<IEventBus>(eventBus);
            container.RegisterInstance<ICommandHandlerRegistry>(commandProcessor);
            container.RegisterInstance(commandProcessor);
            container.RegisterInstance<IEventHandlerRegistry>(eventProcessor);
            container.RegisterInstance(eventProcessor);
#endif
```

> **MarkusPersona:** The code sample also shows how the application uses
> the [Unity Application Block][unity] dependency injection container.  
> The solution only uses Unity in the web application, not in the domain
> classes.  
> For more information, see [Technologies Used in the Reference
> Implementation][r_chapter9] in the Reference Guide.

# Testing 

Describe any special considerations that relate to testing for this bounded context.  

[j_chapter4]:	        Journey_04_ExtendingEnhancing.markdown
[r_chapter9]:     		Reference_09_Technologies.markdown

[inductiveui]:			http://msdn.microsoft.com/en-us/library/ms997506.aspx
[metroux]:              http://msdn.microsoft.com/en-us/library/windows/apps/hh465424.aspx
[unity]:				http://msdn.microsoft.com/en-us/library/ff647202.aspx