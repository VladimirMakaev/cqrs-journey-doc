# Chapter 7: Adding Resilience and Optimizing Performance 

*Revisiting the infrastructure and applying some lessons learned*

# Adding resilience and adding features

The two primary goals for this last stage in our journey are to make the 
system more resilient to failures and to improve the responsiveness of 
the UI. The focus of the effort to harden the system is on the 
**RegistrationProcessManager** class in the Orders and Registrations bounded 
context. The focus on performance is on the way the UI interacts with 
the domain-model during the order creation process. 

[Possibly describe incorporating two other bounded contexts.]

## Working definitions for this chapter 

The following definitions are used for the remainder of this chapter. 
For more detail, and possible alternative definitions, see [A CQRS/ES 
Deep Dive][r_chapter4] in the Reference Guide. 

### Command

A command is a request for the system to perform an action that changes 
the state of the system. Commands are imperatives, for example 
**MakeSeatReservation**. In this bounded context, commands originate 
either from the UI as a result of a user initiating a request, or from 
a process manager when the process manager is directing an aggregate to perform an 
action. 

Commands are processed once by a single recipient. A command bus 
transports commands that command handlers then dispatch to aggregates. 
Sending a command is an asynchronous operation with no return value. 

[TODO: Verify whether they are still async]

### Event

An event, such as **OrderConfirmed**, describes something that has 
happened in the system, typically as a result of a command. Aggregates 
in the domain model raise events. 

Multiple subscribers can handle a specific event. Aggregates publish 
events to an event bus; handlers register for specific types of event on 
the event bus and then deliver the events to the subscriber. In this 
bounded context, the only subscriber is a process manager. 

## User stories 

The team implemented the following user stories during this phase of the 
project.

### Support for discounts to seat prices

Registrants should be able to obtain discounts through the use of 
**Promotional Codes**. A Registrant can enter a **Promotional Code** 
during the ordering process and the system will adjust the total cost of 
the order appropriately. 

The discount associated with an individual **Promotional Code** can be 
from 1% to 100%. Each **Promotional Code** has an associated quota that 
determines how many seats are available at the dicounted price (this may 
be unlimited) and a scope that determines which seat type the code is 
associated with; this can also apply to all seat types rather than just 
one. 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
    Updated to include details of single-use codes and cummulative codes.
  </span>
</div>

### Story 1


### Story 2


### Story 3


## Architecture 

What are the key architectural features? Server-side, UI, multi-tier, cloud, etc. 

# Patterns and concepts 

During this stage of the journey the team looked at options for 
hardening the **RegistrationProcessManager** class. This part of the Orders 
and Registrations bounded context is responsible for managing the 
interactions between the aggregates in the Orders and Registrations 
bounded context and for ensuring that they are all consistent with each 
other. It is important that this process manager is resilient to a wide range 
of failure conditions if the bounded context as a whole is to maintain 
its consistent state. 

When the team tested the V2 release, we discovered that sometimes the 
UI is waiting for the domain to to complete its processing, and for the 
read-models to receive data from the write-model before it can display 
the next screen to the Registrant. 

[Add some stats from test here]

To address this issue, the team identified two possible optimizations: 
optimizing the interaction between the UI and the domain and optimizing 
the command handling process. They decided to address the interaction 
between the UI and the domain first and then to evaluate whether any 
further optimization was necessary. 

## Making the RegistrationProcess class more resilient to failure

Typically, a process manager receives incoming events, and then 
based on the state of the process manager, sends out one or more commands to 
aggregates within the bounded context. When a process manager 
sends out commands, it typically changes its own state. 

The Orders and Registrations bounded context contains the 
**RegistrationProcessManager** class. This process manager is 
responsible for coordinating the activities of the aggregates in both 
this bounded context and the Payments bounded context by routing events 
and commands between them. The process manager is therefore responsible for 
ensuring that the aggregates in these bounded contexts are correctly 
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
  saved its new state without sending out the expected commands. The
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
  
These scenarios can be summarized to identify two issues to address:

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

Instead of making the process manager idempotent, you could ensure that all the 
commands that the process manager sends are idempotent. Restarting the process manager 
may result in sending commands a second time, but if those commands are 
idempotent this will have no adverse affect on the process or the 
system. For this approach to work, you will still need to modify the 
process manager to guarantee that it sends all commands at least once. If the 
commands are idempotent, it doesn't matter if they are sent multiple 
times, but it does matter if a command is never sent at all. 

In the V1 release, most message handling is already either idempotent, 
or the system detects duplicate messages and sends them to a dead-letter 
queue. The exceptions are the **OrderPlaced** event and the 
**SeatsReserved** event. 

### Ensuring that commands are always sent

To ensure that the system always sends commands when the 
**RegistrationProcessManager** class saves its state requires transactional 
behavior. This requires the team to implement a psuedo-transaction 
because it is not possible to enlist the Windows Azure Service Bus in a 
distributed transcation. 

The solution adopted by the team for the V3 release ensures that the 
system persists any commands that the **RegistrationProcessManager** tries to 
send but that fail. When the system next reloads the 
**RegistrationProcessManager** class, it tries to re-send the failed 
commands. If this fails, then the process manager cannot be loaded and cannot 
process any further messages until the cause of the failure is resolved. 

## Optimizing the interactions between the ui and the domain

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
**RegisterToConference** command. 

During testing, the team observed that sometimes the Registrant had to 
wait for the Registrant screen to display while the system performed the 
validation and processed all the commands and events. Behind the scenes, 
the MVC controller waits until a priced order appears in the read-model 
(this indicates that all of the processing is complete) before 
displaying the Registrant screen. 

### Options to reduce the delay in the ui

The team discussed with the domain expert whether or not is always 
necessary to validate the seats availability before the UI sends the 
**RegisterToConference** conference command to the domain. 

> **GaryPersona:** This scenario illustrates some practical issues in
> relation to eventual consistency. The read-side, in this case the
> priced order view model, is eventually consistent with the write-side.
> Typically, when you implement the CQRS pattern you should be able
> embrace eventual consistency and not need to wait in the UI for
> changes to propagate to the read-side. However in this case, the UI
> must wait for the write-model to propagate information that relates to
> a specific order to the read-side. This perhaps indicates a problem
> with the original analysis and design of this part of the system.

The domain expert was clear that the system should confirm that seats 
are available before taking payment. Contoso does not want to sell seats 
and then have to explain to a Registrant that those seats are not in 
fact available. Therefore the team looked for ways to streamline getting 
as far as the Payment screen in the UI. 

> **BethPersona:** This cautious strategy is not appropriate in all
> scenarios. In some cases the business may prefer to take the money
> even if it cannot immediately fulfill the order: the business may know
> that the stock will be replenished soon, or that the customer is happy
> to wait. In this scenario, although Contoso could refund the money to
> a Registrant, the Registrant may purchase flight tickets that are not
> refundable in the belief that the conference registration is
> confirmed. This type of decision is clearly one for the business and
> the domain expert.

#### Optimization #1

Most of the time, there are plenty of seats available for a conference 
and Registrants are not competing for the last few that are available. 
It is only for a brief time that Registrants are competing for the last 
few seats. 

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

The current implementation uses the same messaging infrastructure for 
both commands and events. The Windows Azure Service Bus provides a 
reliable, performant, and scalable infrastructure for messaging in 
Windows Azure. The team plans to evaluate whether the same 
infrastructure is necesssary for all commands and events within the 
Contoso Conference Management System. 

The system uses events as its primary mechanism for integrating between 
bounded contexts. One bounded context can raise an event that is then 
handled in another bounded context. These different bounded contexts 
typically run in different role instances in Windows Azure: for example 
the Conference Management bounded context runs in its own web role and 
integrates with the Orders and Registrations bounded context. The 
Windows Azure Service Bus provides a mechanism to transport messages 
between these worker role instances. 

> **GaryPersona:** It's also possible in the future that for some
> bounded contexts, the read-model will be hosted in a separate role
> instance from the write-model. Windows Azure Service Bus will
> transport the events that the system uses to construct the
> denormalized read-model.

There are a number of factors that the team will consider when we 
determine whether to continue using the Windows Azure Service Bus for 
transporting all command messages. 

* In a CQRS implementation, commands are typically used within a bounded
  context, not between bounded contexts. This may mean that some
  commands only exist within a process (or role instance in Windows
  Azure) and could therefore be handled in memory without the need for
  a messaging infrastructure to transport commands across process
  boundaries.
* How would handling commands in memory instead of using a messaging
  infrastructure affect the resilience of the system in the face of
  failures?
* What performance benefits would actually be gained by handling]
  commands in memory?
* If you treat commands as a two-way, synchronous messaging pattern how
  is this supported by the two alternatives?

> An asynchronous command doesn't exist, it's actually another event. If
> I must accept what you send me and raise an event if I disagree, it's
> no longer you telling me to do something, it's you telling me
> something has been done. This seems like a slight difference at first,
> but it has many implications.  
> Greg Young - Why lot's of developers use one-way command messaging 
> (async handling) when it's not needed? - DDD/CQRS Google Groups

Before implementing this optimization, the team plans to evaluate the 
impact of the optimizations to the interaction between the UI and the 
domain. 

## Scaling out

A further optimization that the team considered was to scale out the 
view model generators that populate the various read-models in the 
system. Every web-role that hosts a view model generator instance must 
handle the events published by the write-side by creating a subscription 
the the Windows Azure Service Bus topics. 

# Implementation details 

This section describes some of the significant features of the 
implementation of the Orders and Registrations bounded context. You may 
find it useful to have a copy of the code so you can follow along. You 
can download a copy of the code from the [Download center][downloadc], 
or check the evolution of the code in the repository on github: 
[mspnp/cqrs-journey-code][repourl]. 

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

### Creating a psuedo transaction when the process manager saves its state and sends a command

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
> not able to load it.

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

If there are not enough available seats, the controller redisplays the current screen, displaying the currently available seat quantities to enable the Registrant to revise her order.

This remaining part of the change is in the **SpecifyRegistrantAndPaymentDetails** method in the **RegistrationController** class. The following code sample from the V2 release shows how before the optimization the controller calls the **WaitUntilSeatsAreConfirmed** method before continuing to the Registrant screen:

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

The following code sample shows the V3 version of this method that no longer waits for the reservation to be confirmed:

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
The second optimization is to perform the calculation of the order total earlier in the process. In the previous code sample, the **SpecifyRegistrantAndPaymentDetails** method still calls the **WaitUntilOrderIsPriced** method which pauses the UI flow until the system calculates an order total and makes it available to the controller by saving it in the priced order view model on the read-side.

The key change to implement this is in the **Order** aggregate. The constructor in the **Order** class now invokes the **CalculateTotal** method and raises an **OrderTotalsCalculated** method as shown in the following code sample:

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

Previously, in the V2 release the **Order** aggregate waited until it received a **MarkAsReserved** command before it called the **CalculateTotal** method.

## Handling commands synchronously and inline

In the V2 release, the system used the Windows Azure Service Bus to deliver all commands to their recipients. This meant that the system delivered the commands asynchronously. In the v3 release, the MVC controllers now send their commands synchronously and inline in order to improve the response times in the UI by bypassing the command bus and delivering commands directly to their handlers.

The team implemeted this behavior by adding the **SynchronousCommandBusDecorator** and **CommandDispatcher** classes to the infrastructure and registering them during the start up of the web role as shown in the following code sample from the **OnCreateContainer** method in the Global.asax.Azure.cs file:

```Cs
var commandBus = new CommandBus(new TopicSender(settings.ServiceBus, "conference/commands"), metadata, serializer);
var synchronousCommandBus = new SynchronousCommandBusDecorator(commandBus);

container.RegisterInstance<ICommandBus>(synchronousCommandBus);
container.RegisterInstance<ICommandHandlerRegistry>(synchronousCommandBus);


container.RegisterType<ICommandHandler, OrderCommandHandler>("OrderCommandHandler");
container.RegisterType<ICommandHandler, ThirdPartyProcessorPaymentCommandHandler>("ThirdPartyProcessorPaymentCommandHandler");
container.RegisterType<ICommandHandler, SeatAssignmentsHandler>("SeatAssignmentsHandler");
```

The following code sample shows how the **SynchronousCommandBusDecorator** class implements sending a command message:

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

Notice how this class tries to send the command synchronously without using the service bus, but if it cannot find a handler for the command, it reverts to using the service bus. The following code sample shows how the **CommandDispatcher** class tries to locate a handler and deliver a command message:

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

## Other optimizations

This section describes some of the other optimizations that the team included in the V3 release.

### Using Prefetch with Windows Azure Service Bus

The team enabled the prefetch option whem the system retrieves messages from the Windows Azure Service Bus. This option enables the system to retrieve multiple messages in a single round-trip to the server and helpes to reduce the latency in retrieving existing messages from the Service Bus topics.

The following code sample from the **SubscriptionReceiver** class ahows how to enable this option.

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

In the V2 release, the **SessionSubscriptionReceiver** creates sessions to receive messages from the Windows Azure Service Bus in sequence. In the V3 release, the **SessionSubscriptionReceiver** creates multiple sessions in parallel. This helps to improve the throughput and reduce the latency when the system retrieves messages from the Service Bus. The follwing code sample shows the new version of the **ReceiveMessages** method in the **SessionSubscriptionReceiver** class.

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

### Task support in MVC 4

As part of the V3 release the team upgraded the Conference.Web.Public 
site to MVC 4. This enabled them to update the 
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

[j_chapter4]:        Journey_04_ExtendingEnhancing.markdown

[repourl]:           https://github.com/mspnp/cqrs-journey-code
[watin]:             http://watin.org
[codefirst]:         http://msdn.microsoft.com/en-us/library/gg197525(VS.103).aspx
[downloadc]:         http://NEEDFWLINK

