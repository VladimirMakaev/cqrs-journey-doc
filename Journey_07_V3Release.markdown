# Chapter 7: Adding Resilience and Optimizing Performance 

*Revisiting the infrastructure and applying some lessons learned*

# Introduction

The two primary goals for this last stage in our journey are to make the 
system more resilient to failures and to improve the responsiveness of 
the UI. The effort to harden the system focuses on the 
**RegistrationProcessManager** class in the Orders and Registrations 
bounded context. The focus on performance is on the way the UI interacts 
with the domain-model during the order creation process. 

## Working definitions for this chapter 

The following definitions are used for the remainder of this chapter. 
For more detail, and possible alternative definitions, see [A CQRS/ES 
Deep Dive][r_chapter4] in the Reference Guide. 

### Command

A command is a request for the system to perform an action that changes 
the state of the system. Commands are imperatives, for example 
**MakeSeatReservation**. In this bounded context, commands originate 
either from the UI as a result of a user initiating a request, or from 
a process manager when the process manager is directing an aggregate to 
perform an action. 

Commands are processed once by a single recipient. Commands are either 
transported to their recipients by a command bus, or delivered directly 
inline. If a command is delivered through a command bus, then then the 
command is sent asynchronously. If the comand can be delivered directly 
inline, then the command is sent synchronously. 

### Event

An event, such as **OrderConfirmed**, describes something that has 
happened in the system, typically as a result of a command. Aggregates 
in the domain model raise events. 

Multiple subscribers can handle a specific event. Aggregates publish 
events to an event bus; handlers register for specific types of event on 
the event bus and then deliver the events to the subscriber. In this 
bounded context, the only subscriber is a process manager.

### Snapshots

Snapshots are an optimization that you can apply to event sourcing: 
instead of replaying all of the persisted events associated with an 
aggregate when it is re-hydrated, you load a recent copy of the state of 
the aggregate and then replay only the events that were persisted after 
saving the snapshot. In this way you can reduce the amount of data that 
you must load from the event store. 

## Architecture 

The application is designed to deploy to Windows Azure. At this stage in 
the journey, the application consists of web roles that contains the 
ASP.NET MVC web applications and a worker role that contains the message 
handlers and domain objects. The application uses SQL Database databases 
for data storage, both on the write-side and the read-side. The 
application uses the Windows Azure Service Bus to provide its messaging 
infrastructure. Figure 1 shows this high-level architecture.

![Figure 1][fig1] 

While you are exploring and testing the solution, you can run it 
locally, either using the Windows Azure compute emulator or by running 
the MVC web application directly and running a console application that 
hosts the handlers and domain objects. When you run the application 
locally, you can use a local SQL Express database instead of SQL Database, 
and use a simple messaging infrastructure implemented in a SQL Express 
database. 

For more information about the options for running the application, see 
[Appendix 1][appendix1]. 

# Patterns and concepts 

During this stage of the journey the team looked at options for 
hardening the **RegistrationProcessManager** class. This part of the 
Orders and Registrations bounded context is responsible for managing the 
interactions between the aggregates in the Orders and Registrations 
bounded context and for ensuring that they are all consistent with each 
other. It is important that this process manager is resilient to a wide 
range of failure conditions if the bounded context as a whole is to 
maintain its consistent state. 

We also ran performance tests using the [Visual Studio Load Test Feature 
Pack][loadtest] to analyze response times and identify bottlenecks. The 
team used Visual Studio Load Test to simulate different numbers of users 
accessing the application, and added additonal tracing into the code to 
record timing information for detailed analysis. As a result of this 
exercise, the team made a number of changes to the system to optimize 
its performance.

> **GaryPersona:** Although in this journey the team did their
> performance testing and optimization work at the end of the project,
> it typically makes sense to do this work as you go, addressing
> performance issues and adding hardening as soon as possible.

> **MarkusPersona:** Because implementing the CQRS pattern leads to a
> very clear separation of responsibities for the many different parts
> that make up the system, we found it relatively easy to add
> optimizations and hardening because many of the necessary changes were
> very localized within the system.

## Making the RegistrationProcessManager class more resilient to failure

Typically, a process manager receives incoming events and then, based on 
the state of the process manager, sends out one or more commands to 
aggregates within the bounded context. When a process manager sends out 
commands, it typically changes its own state. 

The Orders and Registrations bounded context contains the 
**RegistrationProcessManager** class. This process manager is 
responsible for coordinating the activities of the aggregates in both 
this bounded context and the Payments bounded context by routing events 
and commands between them. The process manager is therefore responsible 
for ensuring that the aggregates in these bounded contexts are correctly 
synchronized with each other. 

> **GaryPersona:** An aggregate determines the consistency boundaries
> within the write-model with respect to the consistency of the data
> that the system persists to storage. The process manager manages
> the relationship between different aggregates, possibly in different
> bounded contexts, and ensures that the aggregates are eventually
> consistent with each other.

A failure in the registration process could have adverse consequences 
for the system: the aggregates could get out of synchronization with 
each other which may cause unpredicatable behavior in the system; some 
processes might end up as zombie processes continuing to run and use 
resources but never completing. The team identified the following 
specific failure scenarios related to the **RegistrationProcessManager** 
process manager. The process manager could: 

* Crash or be unable to persist its state after it receives an event but
  before it sends any commands. The message processor may not be able to
  mark the event as complete, so after a timeout the event is placed
  back in the topic subscription and re-processed.
* Crash after it persists its state but before it sends any commands.
  This puts the system into an inconsistent state because the process manager
  saves its new state without sending out the expected commands. The
  original event is put back in the topic subscription and 
  re-processed.
* Fail to mark that an event has been processed. The process manager will
  process the event a second time because after a timeout, the system
  will put the event back onto the service bus topic subscription.
* Timeout while it waits for a specific event that it is expecting. The
  process manager cannot continue processing and reach an expected end-state.
* Receive an event that it does not expect to receive while the process manager
  is in a particular state. This may indicate a problem elsewhere that
  implies that it is unsafe for the process manager to continue.
  
These scenarios can be summarized to identify two specific issues to
address:

1. The **RegistrationProcessManager** handles an event successfully but fails
   to mark the message as complete. The **RegistrationProcessManager** will
   then process the event again after it is automatically returned to
   the subscription to the Windows Azure Service Bus topic.
2. The **RegistrationProcessManager** handles an event successfully, marks it
   as complete, but then fails to send out the commands.

### Making the system resilient whan an event is reprocessed

If the behavior of the process manager itself is idempotent, then if it 
receives and processes an event a second time then this does not result 
in any inconsistencies within the system. This approach would handle the 
first three failure conditions. After a crash, you can restart the 
process manager and reprocess the incoming event a second time. 

Instead of making the process manager idempotent, you could ensure that 
all the commands that the process manager sends are idempotent. 
Restarting the process manager may result in sending commands a second 
time, but if those commands are idempotent this will have no adverse 
affect on the process or the system. For this approach to work, you 
still need to modify the process manager to guarantee that it sends all 
commands at least once. If the commands are idempotent, it doesn't 
matter if they are sent multiple times, but it does matter if a command 
is never sent at all. 

In the V1 release, most message handling is already either idempotent, 
or the system detects duplicate messages and sends them to a dead-letter 
queue. The exceptions are the **OrderPlaced** event and the 
**SeatsReserved** event, so the team modified the way that the V3
release of the system processes these two commands in order to address
this issue.

### Ensuring that commands are always sent

To ensure that the system always sends commands when the 
**RegistrationProcessManager** class saves its state requires 
transactional behavior. This requires the team to implement a 
pseudo-transaction because it is not possible to enlist the Windows 
Azure Service Bus and a SQL Database table together in a distributed
transcation. 

The solution adopted by the team for the V3 release ensures that the 
system persists any commands that the **RegistrationProcessManager** 
tries but fails to send. When the system next reloads the 
**RegistrationProcessManager** class, it tries to re-send the failed 
commands. If this fails, then the process manager cannot be loaded and 
cannot process any further messages until the cause of the failure is 
resolved. 

## Optimizing the interactions between the UI and the domain

The performance tests we ran using Visual Studio Load Test uncovered 
unacceptable response times for Registrants creating orders when the 
system was under load. The intitial optimization effort focused on how
the UI interacts with the domain, and we identified ways to streamline
this aspect of the system.

When a Registrant creates an order, she visits the following sequence of 
screens in the UI. 

1. The Register screen. This screen displays the ticket types for the
   conference, the number of seats currently available. The Registrant
   selects the quantities of each seat type that she would like to
   purchase.
2. The Registrant screen. This screen displays a summary of the order
   that includes a total price and a countdown timer that tells the
   Registrant how long the seats will remain reserved. The Registrant
   enters her details and preferred payment method.
3. The Payment screen. This simulates a third-party payment processor.
4. The Registration success screen. This displays if the payment
   succeeded. It displays to the Registrant an order locator code and
   link to a screen that enables the Registrant to assign Attendees to
   seats.
   
See the section Task-based UI in chapter 5, [Preparing for the V1
Release][j_chapter5] for more information about the screens and flow in
the UI.
   
In the V2 release, the system must process the following commands and 
events between the Register screen and the Registrant screen: 

* RegisterToConference
* OrderPlaced
* MakeSeatReservation
* SeatsReserved
* MarkSeatsAsReserved
* OrderReservationCompleted
* OrderTotalsCalculated

In addition, the MVC controller is also validating that there are 
sufficient seats available to fulfill the order before it sends the 
initial **RegisterToConference** command. 

The team load tested the application using Visual Studio Load Test with 
different user load patterns. We noticed that with a constant load 
pattern of ten virtual users, the UI often has to wait for the domain to 
to complete its processing and for the read-models to receive data from 
the write-model, before it can display the next screen to the 
Registrant. In particular, with the V2 release deployed to medium-sized 
web and worker role instances we found that: 

* With a constant load pattern of a single virtual user creating orders,
  all orders are processed within a five second window.
* With  a constant load pattern of ten virtual users simultaneoulsy
  creating orders, many orders are not processed within the five second
  window.
* With a constant load pattern of ten virtual users simultaneoulsy
  creating orders, the role instances are used sub-optimally (for
  example CPU usage is low).

> **Note:** The five second window is the maximum duration that we want
> to see between the time that the UI sends the initial command on the
> service bus and the time when the priced order becomes visible in the
> read-model enabling the UI to display the next screen to the user.

To address this issue, the team identified two possible optimizations: 
optimizing the interaction between the UI and the domain, and optimizing 
the command handling process. We decided to address the interaction 
between the UI and the domain first and then to evaluate whether any 
further optimization was necessary. In the end, we implemented both of
these optimizations.

### Options to reduce the delay in the UI

The team discussed with the domain expert whether or not is always 
necessary to validate the seats availability before the UI sends the 
**RegisterToConference** command to the domain. 

> **GaryPersona:** This scenario illustrates some practical issues in
> relation to eventual consistency. The read-side, in this case the
> priced order view model, is eventually consistent with the write-side.
> Typically, when you implement the CQRS pattern you should be able
> embrace eventual consistency and not need to wait in the UI for
> changes to propagate to the read-side. However in this case, the UI
> must wait for the write-model to propagate information that relates to
> a specific order to the read-side. This perhaps indicates a problem
> with the original analysis and design of this part of the system.

The domain expert was clear that the system should confirm that seats are 
available before taking payment. Contoso does not want to sell seats and 
then have to explain to a Registrant that those seats are not in fact 
available. Therefore, the team looked for ways to streamline getting as 
far as the Payment screen in the UI. 

> **BethPersona:** This cautious strategy is not appropriate in all
> scenarios. In some cases the business may prefer to take the money
> even if it cannot immediately fulfill the order: the business may know
> that the stock will be replenished soon, or that the customer will be
> happy to wait. In our scenario, although Contoso could refund the
> money to a Registrant if tickets turned out not to be available, a 
> Registrant may decide to purchase flight tickets that are not
> refundable in the belief that the conference registration is
> confirmed. This type of decision is clearly one for the business and
> the domain expert.

#### Optimization #1

Most of the time, there are plenty of seats available for a conference 
and Registrants do not have to compete with each other to reserve seats. 
It is only for a brief time, as the conference beomes close to selling 
out, that Registrants do end up competing for the last few available 
seats. 

If there are plenty of available seats for the conference then there is 
a minimal risk that a Registrant will get as far as the Payment screen 
only to find that the system could not reserve the seats. In this case, 
some of the processing that the V2 release performs before getting to 
the Registrant screen can be allowed to happen asynchronously while the 
Registrant is entering information on the Registrant screen. This 
reduces the chance that the Registrant experiences a delay before seeing 
the Registrant screen. 

However, if the controller checks and finds that there are not enough 
seats available to fulfill the order _before_ it sends the 
**RegisterToConference** command, it can re-display the Register screen 
to enable the Registrant to update her order based on current 
availability. 

> **JanaPersona:** A possible enhancement to this strategy is to look at
> whether there are _likely to be_ enough seats available before sending
> the **RegisterToConference** command. This could reduce the number of
> occassions that a Registrant has to adjust her order as the last few
> seats are sold. However, this scenario will occur infrequently enough
> that it is probably not worth implementing.

#### Optimization #2

In the V2 release, the MVC controller cannot display the Registrant 
screen until the domain publishes the **OrderTotalsCalculated** event 
and the system updates the priced order view model. This event is the 
last event that occurs before the controller can display the screen. 

If the system calculates the total and updates the priced order view 
model earlier, the controller can display the Registrant screen sooner. 
The team determined that the **Order** aggregate could calculate the 
total when the order is placed instead of when the reservation is 
complete. This will enable the UI flow to move more quickly to the 
Registrant screen than in the V2 release.

## Optimizing command processing 

The analysis of the performance data collected from the load tests also 
identified a potential bottleneck in the way that the system delivers 
command messages. The team identified a number of optimizations to make 
in this area. 

The current implementation uses the same messaging infrastructure, the 
Windows Azure Service Bus, for both commands and events. The team plans 
to evaluate whether the same infrastructure is necesssary for all 
commands and events within the Contoso Conference Management System. 

The system uses events as its primary mechanism for integrating between 
bounded contexts. One bounded context can raise an event that is then 
handled in another bounded context. These different bounded contexts 
typically run in different role instances in Windows Azure: for example 
the Conference Management bounded context runs in its own web role and 
integrates with the Orders and Registrations bounded context. The 
Windows Azure Service Bus provides a mechanism to transport messages 
between these worker role instances. 

> **GaryPersona:** It's also possible in the future that, for some
> bounded contexts, the read-model will be hosted in a separate role
> instance from the write-model. Windows Azure Service Bus will
> transport the events that the system uses to construct the
> denormalized read-model.

There are a number of factors that the team will consider when we 
determine whether to continue using the Windows Azure Service Bus for 
transporting all command messages. 

* Which commands, if any, can be processed in memory?
* Will the system lose any resilience if it handles some commands in memory?
* Will there be any significant performance gains if it handles some commands in memory?

In addition, by processing commands in-process, it becomes easier to use
commands as synchronous, two-way messages.

> An asynchronous command doesn't exist, it's actually another event. If
> I must accept what you send me and raise an event if I disagree, it's
> no longer you telling me to do something, it's you telling me
> something has been done. This seems like a slight difference at first,
> but it has many implications.  
> Greg Young - Why lot's of developers use one-way command messaging 
> (async handling) when it's not needed? - DDD/CQRS Google Groups

The team evaluated the impact of the optimizations to the interaction 
between the UI and the domain first, and when those optimizations didn't 
produce the necessary improvement we then implemented this 
optimization. 

## Using snapshots with event sourcing

The performance tests also uncovered a bottleneck in the use of the 
**SeatsAvailability** aggregate. This proved to be the most effective 
optimization when the team implemented it and re-ran the performance 
tests. 

> **JanaPersona:** Once the team identified this bottleneck, it was easy
> to implement and test this solution. One of the advantages of the
> approach we followed implementing the CQRS pattern is that we can make
> small localized changes in the system. Updates don't require us to
> make complex changes across multiple parts of the system.

When the system re-hydrates an aggregate instance from the event store, 
it must load and replay all of the events associated with that aggregate 
instance. A possible optimization here is to store a rolling snapshot of 
the state of the aggregate at some recent point in time so that the 
system only needs to load the snapshot and the subsequent events, 
thereby reducing the number of events that it must reload and replay. 
The only aggregate in the Contoso Conference Management System that is 
likely to accumulate a significant number of events over time is the 
**SeatsAvailability** aggregate. We decided to use the 
[Memento][memento] pattern as the basis of the snapshot solution to use 
with the **SeatAvailability** aggregate. The solution we implmented uses 
a memento to capture the state of the **SeatAvailability** aggregate, 
and then keeps a copy of the memento in a cache. The system then tries 
to work with the cached data instead of always reloading the aggregate 
from the event store. 

## Other optimizations

The team performed some additional optimizations that are listed in the 
"Implementation details" section below. The primary goal of the team 
during this stage of the journey was to optimize the system to ensure 
that the UI appears sufficiently responsive to the user. There are 
additional optimizations that we could perform that would help to 
further improve performance and to optimize the way that the system uses 
resources. For example: a further optimization that the team considered 
was to scale out the view model generators that populate the various 
read-models in the system. Every web-role that hosts a view model 
generator instance must handle the events published by the write-side by 
creating a subscription the the Windows Azure Service Bus topics. 

## No downtime migration

The team planned to have a no downtime migration from the V2 to the V3 
release in Windows Azure. Although the web roles continue to run during 
the migration, it is still necessary for this migration to pause the 
worker role: this means that during the migration process, any 
Registrants in the middle of creating an order will see the message 
"Cannot determine the state of the registration" until the migration 
completes. 

The following list summarizes the steps of the migration:

1. Deploy the V3 release to the staging slot in Windows Azure.
2. Switch the V2 release worker role for the V3 release worker role.
   While this is taking place, Registrants continue to use the V2 web
   roles and see the "Cannot determine the state of the registration"
   message because for a while, no worker role instance is available.
3. Switch the V2 release web role for the V3 release worker role.

For details of these steps, see Appendix 1,
"[Building and Running the Sample Code][appendix]."

The reason that the worker role is unavailable for a period of time is 
that the V2 release worker role must have its **MaintenanceMode** 
property set to **true** before it is safe to enable the V3 worker role, 
and setting this property requires Windows Azure to recycle the worker 
role. 

> **JanaPersona:** A possible solution to make this a true no-downtime
> migration is to use a message-based approach that uses a configuration
> bus.

### Data migration

The V3 release uses one additional table called 
**UndispatchedMessages**, and requires an additional column called 
**ReservationExpirationDate** in the **PricedOrders** table. 

The system creates these if they don't already exist when the V3 release 
worker starts up by using Entity Framework database initializers. See 
the **MigrationToV3** project in the **Conference** solution for more 
details. 

# Implementation details 

This section describes some of the significant features of the 
implementation of the Orders and Registrations bounded context. You may 
find it useful to have a copy of the code so you can follow along. You 
can download a copy of the code from the [Download center][downloadc], 
or check the evolution of the code in the repository on github: 
[mspnp/cqrs-journey-code][repourl]. You can download the code from the
V3 release from the [Tags][tags] page on Github.

> **Note:** Do not expect the code samples to exactly match the code in
> the reference implementation. This chapter describes a step in the
> CQRS journey, the implementation may well change as we learn more and
> refactor the code.

## Hardening the RegistrationProcessManager class

This section describes how the team hardened the **RegistrationProcessManager** 
process manager by checking for duplicate instances of the **SeatsReserved** 
and **OrderPlaced** messages. 

### Detecting duplicate SeatsReserved events

Typically, the **RegistrationProcessManager** class sends a 
**MakeSeatReservation** command to the **SeatAvailability** aggregate, 
the **SeatAvailability** aggregate publishes a **SeatsReserved** event 
when it has made the reservation, and the **RegistrationProcessManager** 
receives this notification. In the V2 release, the **SeatsReserved** 
event is not idempotent and the **RegistrationProcessManager** could, 
potentially, receive multiple copies of this event. The solution 
described in this section enables the **RegistrationProcessManager** to 
identify any duplicate **SeatsReserved** messages and then ignore them 
instead of re-processing them. 

Before the **RegistrationProcessManager** class sends the 
**MakeSeatReservation** command, it saves the **Id** of the command in 
the **SeatReservationCommandId** variable as shown in the following code 
sample: 

```Cs
public void Handle(OrderPlaced message)
{
    if (this.State == ProcessState.NotStarted)
    {
        this.ConferenceId = message.ConferenceId;
        this.OrderId = message.SourceId;
        // use the order id as the opaque reservation id for the seat reservation
        this.ReservationId = message.SourceId;
        this.ReservationAutoExpiration = message.ReservationAutoExpiration;
        this.State = ProcessState.AwaitingReservationConfirmation;

        var seatReservationCommand =
            new MakeSeatReservation
            {
                ConferenceId = this.ConferenceId,
                ReservationId = this.ReservationId,
                Seats = message.Seats.ToList()
            };
        this.SeatReservationCommandId = seatReservationCommand.Id;
        this.AddCommand(seatReservationCommand);

        ...
}
```

Then, when it handles the **SeatsReserved** event, it checks that the 
**CorrelationId** property of the event matches the most recent value of 
the **SeatReservationCommandId** variable as shown in the following code 
sample: 

```Cs
public void Handle(Envelope<SeatsReserved> envelope)
{
    if (this.State == ProcessState.AwaitingReservationConfirmation)
    {
        if (envelope.CorrelationId != null)
        {
            if (string.CompareOrdinal(this.SeatReservationCommandId.ToString(), envelope.CorrelationId) != 0)
            {
                // skip this event
                Trace.TraceWarning("Seat reservation response for reservation id {0} does not match the expected correlation id.", envelope.Body.ReservationId);
                return;
            }
        }

        ...
}
```

Notice how this **Handle** method handles an **Envelope** instance 
instead of a **SeatsReserved** instance. As a part of the V3 release, 
events are wrapped in an **Envelope** instance that includes the 
**CorrelationId** property. The **DoDispatchMessage** method in the 
**EventDispatcher** assigns the value of the correlation Id. 

> **MarkusPersona:** As a side-effect of adding this feature, the
> **EventProcessor** class can no longer use the **dynamic** keyword
> when it forwards events to handlers. Now in V3 it uses the new
> **EventDispatcher** class: this class uses reflection to identify the
> correct handlers for a given message type.

During performance testing, the team identified a further issue with 
this specific **SeatsReserved** event. Because of a delay elsewhere in 
the system when it was under load, a second copy of the 
**SeatsReserved** event was being published. This **Handle** method was 
then throwing an exception that caused the system to retry processing 
the message several times before sending it to a dead-letter queue. To 
address this specific issue, the team modified this method by adding the 
**else if** clause as shown in the following code sample: 

```Cs
public void Handle(Envelope<SeatsReserved> envelope)
{
    if (this.State == ProcessState.AwaitingReservationConfirmation)
    {
        ...
    }
    else if (string.CompareOrdinal(this.SeatReservationCommandId.ToString(), envelope.CorrelationId) == 0)
    {
        Trace.TraceInformation("Seat reservation response for request {1} for reservation id {0} was already handled. Skipping event.", envelope.Body.ReservationId, envelope.CorrelationId);
    }
    else
    {
        throw new InvalidOperationException("Cannot handle seat reservation at this stage.");
    }
}
```

> **MarkusPersona:** This optimization was only applied for this
> specific message. Notice that it makes use of the value of
> **SeatReservationCommandId** property that was previously saved in the
> instance. If you want to perform this kind of check on other messages
> you'll need to store more information in the process manager.

### Detecting duplicate OrderPlaced events

To achieve this, the **RegistrationProcessRouter** class now performs a 
check to see of the event is has already been processed. The new V3 
version of the code is shown in the following code sample: 

```Cs
public void Handle(OrderPlaced @event)
{
    using (var context = this.contextFactory.Invoke())
    {
        lock (lockObject)
        {
            var process = context.Find(x => x.OrderId == @event.SourceId && x.Completed == false);
            if (process == null)
            {
                // If the process already exists, it means that the OrderPlaced event is being reprocessed because the message 
                // could not be completed. No need to handle it again.
                process = new RegistrationProcess();
                process.Handle(@event);

                context.Save(process);
            }
        }
    }
}
```

### Creating a pseudo transaction when the RegistrationProcessManager class saves its state and sends a command

It is not possible to have a transaction in Windows Azure that spans 
persisting the **RegistrationProcessManager** to storage and sending the 
command. Therefore the team decided to save any failed commands that the 
process manager sends, and to automatically retry those commands the next time 
that the system loads the process manager from storage. 

> **MarkusPersona:** The migration utility for moving to the V3 release
> updates the database schema to accomodate the new storage requirement.

The following code sample from the **SqlProcessDataContext** class shows 
how the system persists the failed commands along with the state of the 
process manager: 

```Cs
public void Save(T process)
{
    var entry = this.context.Entry(process);

    if (entry.State == System.Data.EntityState.Detached)
        this.context.Set<T>().Add(process);

    var commandIndex = 0;
    var commands = process.Commands.ToList();

    try
    {
        for (int i = 0; i < commands.Count; i++)
        {
            this.commandBus.Send(commands[i]);
            commandIndex++;
        }
    }
    catch (Exception) // We catch a generic exception as we don't know what implementation of ICommandBus we might be using.
    {
        var pending = this.context.Set<PendingCommandsEntity>().Find(process.Id);
        if (pending == null)
        {
            pending = new PendingCommandsEntity(process.Id);
            this.context.Set<PendingCommandsEntity>().Add(pending);
        }

        pending.Commands = this.serializer.Serialize(commands.Skip(commandIndex));
    }

    // Saves both the state of the process as well as the pending commands if any.
    this.context.SaveChanges();
}
```

The following code sample from the **SqlProcessDataContext** class shows 
how the system retries the failed commands when the system next reloads 
the process manager: 

```Cs
public T Find(Expression<Func<T, bool>> predicate)
{
    var process = this.context.Set<T>().Where(predicate).FirstOrDefault();
    if (process == null)
        return default(T);

    var pendingCommands = this.context.Set<PendingCommandsEntity>().Find(process.Id);
    if (pendingCommands != null)
    {
        // Must dispatch pending commands before the process 
        // can be further used.
        var commands = this.serializer.Deserialize<IEnumerable<Envelope<ICommand>>>(pendingCommands.Commands).OfType<Envelope<ICommand>>().ToList();
        var commandIndex = 0;

        // Here we try again, one by one. Anyone might fail, so we have to keep 
        // decreasing the pending commands count until no more are left.
        try
        {
            for (int i = 0; i < commands.Count; i++)
            {
                this.commandBus.Send(commands[i]);
                commandIndex++;
            }
        }
        catch (Exception) // We catch a generic exception as we don't know what implementation of ICommandBus we might be using.
        {
            pendingCommands.Commands = this.serializer.Serialize(commands.Skip(commandIndex));
            this.context.SaveChanges();
            // If this fails, we propagate the exception.
            throw;
        }

        // If succeed, we delete the pending commands.
        this.context.Set<PendingCommandsEntity>().Remove(pendingCommands);
        this.context.SaveChanges();
    }

    return process;
}
```

> **Note:** If it is still not possible to send a command when it is
> re-tried. The **Find** method throws an exception and the system is
> not able to load the **RegistrationProcessManager** instance.

## Optimizing the UI

The first optimization is to allow the UI to navigate directly to the 
Registrant screen provided that there are plenty of seats still 
available for the conference. This change is introduced in the 
**StartRegistration** method in the **RegistrationController** class 
that now performs an additional check to verify that there are enough 
remaining seats to stand a good chance of making the reservation before 
it sends the **RegisterToConference** command as shown in the following 
code sample: 

```Cs
[HttpPost]
public ActionResult StartRegistration(RegisterToConference command, int orderVersion)
{
    var existingOrder = orderVersion != 0 ? this.orderDao.FindDraftOrder(command.OrderId) : null;
    var viewModel = existingOrder == null ? this.CreateViewModel() : this.CreateViewModel(existingOrder);
    viewModel.OrderId = command.OrderId;
    
    if (!ModelState.IsValid)
    {
        return View(viewModel);
    }

    // checks that there are still enough available seats, and the seat type IDs submitted ar valid.
    ModelState.Clear();
    bool needsExtraValidation = false;
    foreach (var seat in command.Seats)
    {
        var modelItem = viewModel.Items.FirstOrDefault(x => x.SeatType.Id == seat.SeatType);
        if (modelItem != null)
        {
            if (seat.Quantity > modelItem.MaxSelectionQuantity)
            {
                modelItem.PartiallyFulfilled = needsExtraValidation = true;
                modelItem.OrderItem.ReservedSeats = modelItem.MaxSelectionQuantity;
            }
        }
        else
        {
            // seat type no longer exists for conference.
            needsExtraValidation = true;
        }
    }

    if (needsExtraValidation)
    {
        return View(viewModel);
    }

    command.ConferenceId = this.ConferenceAlias.Id;
    this.commandBus.Send(command);

    return RedirectToAction(
        "SpecifyRegistrantAndPaymentDetails",
        new { conferenceCode = this.ConferenceCode, orderId = command.OrderId, orderVersion = orderVersion });
}
```

If there are not enough available seats, the controller redisplays the 
current screen, displaying the currently available seat quantities to 
enable the Registrant to revise her order. 

This remaining part of the change is in the 
**SpecifyRegistrantAndPaymentDetails** method in the 
**RegistrationController** class. The following code sample from the V2 
release shows how before the optimization the controller calls the 
**WaitUntilSeatsAreConfirmed** method before continuing to the 
Registrant screen: 

```Cs
[HttpGet]
[OutputCache(Duration = 0, NoStore = true)]
public ActionResult SpecifyRegistrantAndPaymentDetails(Guid orderId, int orderVersion)
{
    var order = this.WaitUntilSeatsAreConfirmed(orderId, orderVersion);
    if (order == null)
    {
        return View("ReservationUnknown");
    }

    if (order.State == DraftOrder.States.PartiallyReserved)
    {
        return this.RedirectToAction("StartRegistration", new { conferenceCode = this.ConferenceCode, orderId, orderVersion = order.OrderVersion });
    }

    if (order.State == DraftOrder.States.Confirmed)
    {
        return View("ShowCompletedOrder");
    }

    if (order.ReservationExpirationDate.HasValue && order.ReservationExpirationDate < DateTime.UtcNow)
    {
        return RedirectToAction("ShowExpiredOrder", new { conferenceCode = this.ConferenceAlias.Code, orderId = orderId });
    }

    var pricedOrder = this.WaitUntilOrderIsPriced(orderId, orderVersion);
    if (pricedOrder == null)
    {
        return View("ReservationUnknown");
    }

    this.ViewBag.ExpirationDateUTC = order.ReservationExpirationDate;

    return View(
        new RegistrationViewModel
        {
            RegistrantDetails = new AssignRegistrantDetails { OrderId = orderId },
            Order = pricedOrder
        });
}
```

The following code sample shows the V3 version of this method that no 
longer waits for the reservation to be confirmed: 

```Cs
[HttpGet]
[OutputCache(Duration = 0, NoStore = true)]
public ActionResult SpecifyRegistrantAndPaymentDetails(Guid orderId, int orderVersion)
{
    var pricedOrder = this.WaitUntilOrderIsPriced(orderId, orderVersion);
    if (pricedOrder == null)
    {
        return View("PricedOrderUnknown");
    }

    if (!pricedOrder.ReservationExpirationDate.HasValue)
    {
        return View("ShowCompletedOrder");
    }

    if (pricedOrder.ReservationExpirationDate < DateTime.UtcNow)
    {
        return RedirectToAction("ShowExpiredOrder", new { conferenceCode = this.ConferenceAlias.Code, orderId = orderId });
    }

    return View(
        new RegistrationViewModel
        {
            RegistrantDetails = new AssignRegistrantDetails { OrderId = orderId },
            Order = pricedOrder
        });
}
```
The second optimization is to perform the calculation of the order total 
earlier in the process. In the previous code sample, the 
**SpecifyRegistrantAndPaymentDetails** method still calls the 
**WaitUntilOrderIsPriced** method which pauses the UI flow until the 
system calculates an order total and makes it available to the 
controller by saving it in the priced order view model on the read-side. 

The key change to implement this is in the **Order** aggregate. The 
constructor in the **Order** class now invokes the **CalculateTotal** 
method and raises an **OrderTotalsCalculated** event as shown in the 
following code sample: 

```Cs
public Order(Guid id, Guid conferenceId, IEnumerable<OrderItem> items, IPricingService pricingService)
    : this(id)
{
    var all = ConvertItems(items);
    var totals = pricingService.CalculateTotal(conferenceId, all.AsReadOnly());

    this.Update(new OrderPlaced
    {
        ConferenceId = conferenceId,
        Seats = all,
        ReservationAutoExpiration = DateTime.UtcNow.Add(ReservationAutoExpiration),
        AccessCode = HandleGenerator.Generate(6)
    });
    this.Update(new OrderTotalsCalculated { Total = totals.Total, Lines = totals.Lines != null ? totals.Lines.ToArray() : null, IsFreeOfCharge = totals.Total == 0m });
}
```

Previously, in the V2 release the **Order** aggregate waited until it 
received a **MarkAsReserved** command before it called the 
**CalculateTotal** method. 

## Handling commands synchronously and inline

In the V2 release, the system used the Windows Azure Service Bus to 
deliver all commands to their recipients. This meant that the system 
delivered the commands asynchronously. In the v3 release, the MVC 
controllers now send their commands synchronously and inline in order to 
improve the response times in the UI by bypassing the command bus and 
delivering commands directly to their handlers. In addition, in the 
**ConferenceProcessor** worker role, commands sent to **Order** 
aggregates are sent synchronously inline using the same mechanism. 

> **MarkusPersona:** We still continue to send commands to the
> **SeatsAvailability** aggregate asynchronously because with multiple
> instances of the **RegistrationProcessManager** running in parallel,
> there will contention as multiple threads all try to access the same
> instance of the **SeatsAvailability** aggregate.

The team implemeted this behavior by adding the 
**SynchronousCommandBusDecorator** and **CommandDispatcher** classes to 
the infrastructure and registering them during the start up of the web 
role as shown in the following code sample from the 
**OnCreateContainer** method in the Global.asax.Azure.cs file: 

```Cs
var commandBus = new CommandBus(new TopicSender(settings.ServiceBus, "conference/commands"), metadata, serializer);
var synchronousCommandBus = new SynchronousCommandBusDecorator(commandBus);

container.RegisterInstance<ICommandBus>(synchronousCommandBus);
container.RegisterInstance<ICommandHandlerRegistry>(synchronousCommandBus);


container.RegisterType<ICommandHandler, OrderCommandHandler>("OrderCommandHandler");
container.RegisterType<ICommandHandler, ThirdPartyProcessorPaymentCommandHandler>("ThirdPartyProcessorPaymentCommandHandler");
container.RegisterType<ICommandHandler, SeatAssignmentsHandler>("SeatAssignmentsHandler");
```

> **Note:** There is similar code in the Conference.Azure.cs file to
> configure the worker role to send some commands inline.


The following code sample shows how the 
**SynchronousCommandBusDecorator** class implements sending a command 
message: 

```Cs
public class SynchronousCommandBusDecorator : ICommandBus, ICommandHandlerRegistry
{
    private readonly ICommandBus commandBus;
    private readonly CommandDispatcher commandDispatcher;

    public SynchronousCommandBusDecorator(ICommandBus commandBus)
    {
        this.commandBus = commandBus;
        this.commandDispatcher = new CommandDispatcher();
    }

    ...

    public void Send(Envelope<ICommand> command)
    {
        if (!this.DoSend(command))
        {
            Trace.TraceInformation("Command with id {0} was not handled locally. Sending it through the bus.", command.Body.Id);
            this.commandBus.Send(command);
        }
    }

    ...

    private bool DoSend(Envelope<ICommand> command)
    {
        bool handled = false;

        try
        {
            var traceIdentifier = string.Format(CultureInfo.CurrentCulture, " (local handling of command with id {0})", command.Body.Id);
            handled = this.commandDispatcher.ProcessMessage(traceIdentifier, command.Body, command.MessageId, command.CorrelationId);

        }
        catch (Exception e)
        {
            Trace.TraceWarning("Exception handling command with id {0} synchronously: {1}", command.Body.Id, e.Message);
        }

        return handled;
    }
}
```

Notice how this class tries to send the command synchronously without 
using the service bus, but if it cannot find a handler for the command, 
it reverts to using the service bus. The following code sample shows how 
the **CommandDispatcher** class tries to locate a handler and deliver a 
command message: 

```Cs
public bool ProcessMessage(string traceIdentifier, ICommand payload, string messageId, string correlationId)
{
    var commandType = payload.GetType();
    ICommandHandler handler = null;

    if (this.handlers.TryGetValue(commandType, out handler))
    {
        Trace.WriteLine("-- Handled by " + handler.GetType().FullName + traceIdentifier);
        ((dynamic)handler).Handle((dynamic)payload);
        return true;
    }
    else
    {
        return false;
    }
}
```

> **MarkusPersona:** Notice how we use the **dynamic** keyword when we
> dispatch the command to its registered handler.

## Implementing snapshots with the memento pattern

In the Contoso Conference Management System, the only event-sources 
aggregate that is likely to have a significant number of events and 
benefit from snapshots is the **SeatAvailability** aggregate. 

> **MarkusPersona:** Because we chose to use the memento pattern, the
> snapshot of the aggregate state is stored in the memento.

The following code sample from the **Save** method in the 
**AzureEventSourcedRepository** class shows how the system creates a 
cached memento object if there is a cache and the aggregate implements 
the **IMementoOriginator** interface. 

```Cs
public void Save(T eventSourced, string correlationId)
{
    ...

    this.cacheMementoIfApplicable.Invoke(eventSourced);
}
```

Then, when the system loads an aggregate by invoking the **Find** method 
in the **AzureEventSourcedRepository** class, it checks to see of there 
is a cached memento containing A snapshot of the state of the object to 
use: 

```Cs
public T Find(Guid id)
{
    var memento = this.getMementoFromCache(id);
    if (memento != null)
    {
        // NOTE: if we had a guarantee that this is running in a single process, there is
        // no need to check if there are new events after the cached version.
        var deserialized = this.eventStore.Load(GetPartitionKey(id), memento.Version + 1)
            .Select(this.Deserialize);

        return this.originatorEntityFactory.Invoke(id, memento, deserialized);
    }
    else
    {
        var deserialized = this.eventStore.Load(GetPartitionKey(id), 0)
            .Select(this.Deserialize)
            .AsCachedAnyEnumerable();

        if (deserialized.Any())
        {
            return this.entityFactory.Invoke(id, deserialized);
        }
    }

    return null;
}
```

In this solution, whenever the system updates the aggregate and invokes 
the **Save** method, it also updates the existing memento. Therefore, if 
there is only a single process, the **Find** method doesn't need to load 
events from the event store. However, the Contoso Conference Management 
System may use multiple processes, therefore the **Find** method checks 
in the event store for recent events. If the cached memento expires, the 
**Find** method loads all of the events associated with the aggregate 
from the store. 

**MarkusPersona:** If we were sure that we'd always be running this in a 
single process we could optimize further by not querying for new events. 

The following code sample shows how the **SeatsAvailability** class adds 
a snapshot of its state data to the memento object to be cached: 

```Cs
public IMemento SaveToMemento()
{
    return new Memento
    {
        Version = this.Version,
        RemainingSeats = this.remainingSeats.ToArray(),
        PendingReservations = this.pendingReservations.ToArray(),
    };
}
```

## Other optimizations

This section describes some of the other optimizations that the team 
included in the V3 release. 

### Using Prefetch with Windows Azure Service Bus

The team enabled the prefetch option whem the system retrieves messages 
from the Windows Azure Service Bus. This option enables the system to 
retrieve multiple messages in a single round-trip to the server and 
helpes to reduce the latency in retrieving existing messages from the 
Service Bus topics. 

The following code sample from the **SubscriptionReceiver** class ahows 
how to enable this option. 

```Cs
protected SubscriptionReceiver(ServiceBusSettings settings, string topic, string subscription, RetryStrategy backgroundRetryStrategy)
{
    this.settings = settings;
    this.topic = topic;
    this.subscription = subscription;

    this.tokenProvider = TokenProvider.CreateSharedSecretTokenProvider(settings.TokenIssuer, settings.TokenAccessKey);
    this.serviceUri = ServiceBusEnvironment.CreateServiceUri(settings.ServiceUriScheme, settings.ServiceNamespace, settings.ServicePath);

    var messagingFactory = MessagingFactory.Create(this.serviceUri, tokenProvider);
    this.client = messagingFactory.CreateSubscriptionClient(topic, subscription);
    this.client.PrefetchCount = 40;

    ...
}
```

### Accepting multiple sessions in parallel

In the V2 release, the **SessionSubscriptionReceiver** creates sessions 
to receive messages from the Windows Azure Service Bus in sequence. In 
the V3 release, the **SessionSubscriptionReceiver** creates multiple 
sessions in parallel. This helps to improve the throughput and reduce 
the latency when the system retrieves messages from the Service Bus. The 
following code sample shows the new version of the **ReceiveMessages** 
method in the **SessionSubscriptionReceiver** class. 

```Cs
private void ReceiveMessages(CancellationToken cancellationToken)
{
    while (!cancellationToken.IsCancellationRequested)
    {
        ...

        // starts a new task to process new sessions in parallel if enough threads are available
        Task.Factory.StartNew(() => this.ReceiveMessages(this.cancellationSource.Token), this.cancellationSource.Token);

        while (!cancellationToken.IsCancellationRequested)
        {
            BrokeredMessage message = null;
            try
            {
                try
                {
                    // Long polling is used when accepting session and not here. If there are no messages left in session we continue.
                    message = this.receiveRetryPolicy.ExecuteAction(() => session.Receive(TimeSpan.Zero));
                }
                catch (Exception e)

                ...

                if (message == null)
                {
                    // If we have no more messages for this session, exit to close the session
                    break;
                }

                this.MessageReceived(this, new BrokeredMessageEventArgs(message));
            }
            finally
            {
                ...
            }
        }

        this.receiveRetryPolicy.ExecuteAction(() => session.Close());
        // As we have no more messages for this session, end this task, as there will already at least
        // 1 other tasks polling for new sessions to accept.
        return;
    }
}
```

Because of this change, the team also added an optimistic concurrency 
check when the system saves the **RegistrationProcessManager** class by 
adding a timestamp property to the **RegistrationProcessManager** class 
as shown in the following code sample: 

```Cs
[ConcurrencyCheck]
[Timestamp]
public byte[] TimeStamp { get; private set; }
```

For more information, see [Code First Data Annotations][codefirst] on
the MSDN website.

With the optimistic concurrency check in place, we also removed the C# 
lock in the **SessionSubscriptionReceiver** class that was a potential 
bottleneck in the system. 

### Publishing Events in parallel

In chapter 5, [Preparing for the V1 Release][j_chapter5], you saw how 
the system publishes events whenever it saves them to the event store. 
This optimization enables the system to publish some of these events in 
parallel insteand of publishing them sequentially. It is important that 
the events associated with a specific aggregate instance are sent in the 
correct order, so the system only creates new tasks for different 
partition keys. The following code sample from the **Start** method in 
the **EventStoreBusPublisher** class shows how the parallel tasks are 
defined: 

```Cs
Task.Factory.StartNew(
    () =>
    {
        try
        {
            Parallel.ForEach(
                new BlockingCollectionPartitioner<string>(this.enqueuedKeys),
                new ParallelOptions
                {
                    MaxDegreeOfParallelism = MaxDegreeOfParallelism,
                    CancellationToken = cancellationToken
                },
                this.ProcessPartition);
        }
        catch (OperationCanceledException)
        {
            return;
        }
    },
    TaskCreationOptions.LongRunning);
```

For more information about the **BlockingCollectionPartitioner** class, 
see the blog post [ParallelExtensionsExtras Tour - #4 - 
BlockingCollectionExtensions][parallelext].

### Completing messages asynchronously

The system uses the peek/lock mechanism to retrieve messages from the Service Bus topic subscriptions. The **BrokeredMessageExtensions** class now includes three new methods to support completing messages asynchronously:

* **SafeCompleteAsync**
* **SafeAbandonAsync**
* **SafeDeadLetterAsync**

The system uses these methods when the **ReleaseMessageLockAsynchronously** property is set to **true** in the **CommandProcessor** class as shown in the following code sample from the ConferenceProcessor.Azure.cs file:

```Cs
var commandProcessor =
    new CommandProcessor(new SubscriptionReceiver(azureSettings.ServiceBus, Topics.Commands.Path, Topics.Commands.Subscriptions.All), serializer)
    {
        ReleaseMessageLockAsynchronously = true
    };
```

### Task support in ASP.NET MVC4

As part of the V3 release the team upgraded the Conference.Web.Public 
site to ASP.NET MVC4. This enabled them to update the 
**RegistrationController** MVC controller to use the new task support 
for asynchronous controllers. The following code sample shows an example 
of this: 

```Cs
[HttpGet]
[OutputCache(Duration = 0, NoStore = true)]
public Task<ActionResult> SpecifyRegistrantAndPaymentDetails(Guid orderId, int orderVersion)
{
    return this.WaitUntilOrderIsPriced(orderId, orderVersion)
    .ContinueWith<ActionResult>(

    ...

    );
}

...

private Task<PricedOrder> WaitUntilOrderIsPriced(Guid orderId, int lastOrderVersion)
{
    return
        TimerTaskFactory.StartNew<PricedOrder>(
            () => this.orderDao.FindPricedOrder(orderId),
            order => order != null && order.OrderVersion > lastOrderVersion,
            PricedOrderPollPeriodInMilliseconds,
            DateTime.Now.AddSeconds(PricedOrderWaitTimeoutInSeconds));
}
```

# Impact on testing 

During this stage of the journey the team re-organized the 
**Conference.Specflow** project in the **Conference.AcceptanceTests** 
Visual Studio solution to better reflect the purpose of the tests. 

## Integration tests

The tests in the **Features\Integration** folder in the 
**Conference.Specflow** project are designed to test the behavior of the 
domain directly, verifying the behavior of the domain by looking at the 
commands and events that are sent and received. These tests are designed 
to be understood by "programmers" rather than "domain experts" and are 
formulated using a more technical vocabulary than the ubiquitous 
language. In addition to verifying the behavior of the domain and 
helping developers to understand the flow of commands and events in the 
system, these tests proved to be useful in testing the behavior of the 
domain in scenarios where events are lost or are received out of order. 

The **Conference** folder contains integration tests for the Conference 
Management bounded context, and the **Registration** folder contains 
tests for the Orders and Registrations bounded context.

> **MarkusPersona:** These integration tests make the assumption that
> the command handlers trust the sender of the commands to send valid
> command messages. This may not be appropriate for other systems that
> you may be designing tests for.

## User interface tests

The **UserInterface** folder contains the acceptance tests. These tests 
are described in more detail in [Chapter 4, Extending and Enhancing the 
Orders and Registrations Bounded Context][j_chapter4]. The 
**Controllers** folder contains the tests that use the MVC controllers 
as the point of entry, and the **Views** folder contains the tests that 
use [WatiN][watin] to drive the system through its UI. 


[fig1]:              images/Journey_07_TopLevel.png?raw=true

[j_chapter4]:        Journey_04_ExtendingEnhancing.markdown
[j_chapter5]:        Journey_05_PaymentsBC.markdown
[appendix]:          Appendix1_Running.markdown

[repourl]:           https://github.com/mspnp/cqrs-journey-code
[watin]:             http://watin.org
[codefirst]:         http://msdn.microsoft.com/en-us/library/gg197525(VS.103).aspx
[downloadc]:         http://NEEDFWLINK
[parallelext]:       http://blogs.msdn.com/b/pfxteam/archive/2010/04/06/9990420.aspx
[tags]:              https://github.com/mspnp/cqrs-journey-code/tags
[memento]:           http://www.oodesign.com/memento-pattern.html
[loadtest]:          http://www.microsoft.com/visualstudio/en-us/products/2010-editions/load-test-virtual-user-pack/overview