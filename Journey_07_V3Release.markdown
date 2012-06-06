## Chapter 7
# Adding Resilience and Optimizing Performance 

*Revisiting the infrastructure and applying some lessons learned*

# Adding Resilience and Adding Features

The two primary goals for this last stage in our journey are to make the 
system more resilient to failures and to improve the responsiveness of 
the UI. The focus of the effort to harden the system is on the 
**RegistrationProcess** workflow in the Orders and Registrations bounded 
context. The focus on performance is on the way the UI interacts with 
the domain-model during the order creation process. 

[Possibly describe incorporating two other bounded contexts.]

## Working Definitions for this Chapter 

The following definitions are used for the remainder of this chapter. 
For more detail, and possible alternative definitions, see [A CQRS/ES 
Deep Dive][r_chapter4] in the Reference Guide. 

### Command

A command is a request for the system to perform an action that changes 
the state of the system. Commands are imperatives, for example 
**MakeSeatReservation**. In this bounded context, commands originate 
either from the UI as a result of a user initiating a request, or from 
a workflow when the workflow is directing an aggregate to perform an 
action. 

Commands are processed once by a single recipient. A command bus 
transports commands that command handlers then dispatch to aggregates. 
Sending a command is an asynchronous operation with no return value. 

### Event

An event describes something that has happened in the system, typically 
as a result of a command. Aggregates in the domain model raise events. 

Multiple subscribers can handle a specific event. Aggregates publish 
events to an event bus; handlers register for specific types of event on 
the event bus and then deliver the events to the subscriber. In this 
bounded context, the only subscriber is a workflow. 

## User Stories 

The team implemented the following user stories during this phase of the 
project.

### Support for Discounts to Seat Prices

Registrants should be able to obtain discounts through the use of 
**Promotional Codes**. A registrant can enter a **Promotional Code** 
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

# Patterns and Concepts 

During this stage of the journey the team looked at options for 
hardening the **RegistrationProcess** workflow. This part of the Orders 
and Registrations bounded context is responsible for managing the 
interactions between the aggregates in the Orders and Registrations 
bounded context and for ensuring that they are all consistent with each 
other. It is important that this workflow is resilient to a wide range 
of failure conditions if the bounded context as a whole is to maintain 
its consistent state. 

When the team tested the V2 release, they discovered that sometimes the 
UI is waiting for the domain to to complete its processing, and for the 
read-models to receive data from the write-model before it can display 
the next screen to the Registrant. 

[Add some stats from test here]

To address this issue, the team identified two possible optimizations: 
optimizing the interaction between the UI and the domain and optimizing 
the command handling process. They decided to address the interaction 
between the UI and the domain first and then to evaluate whether any 
further optimization was necessary. 

## Making the RegistrationProcess Workflow More Resilient to Failure

Typically, a coordinating workflow receives incoming events, and then 
based on the state of the workflow, sends out one or more commands to 
aggregates within the bounded context. When a coordinating workflow 
sends out commands, it typically changes its own state. 

The Orders and Registrations bounded context contains the 
**RegistrationProcess** coordinating workflow. This workflow is 
responsible for coordinating the activities of the aggregates in both 
this bounded context and the Payments bounded context by routing events 
and commands between them. The workflow is therefore responsible for 
ensuring that the aggregates in these bounded contexts are correctly 
synchronized with each other. 

> **BharathPersona:** An aggregate determines the consistency boundaries
> within the write-model with respect to the consistency of the data
> that the system persists to storage. The coordinating workflow manages
> the relationship between different aggregates, possibly in different
> bounded contexts, and ensures that the aggregates are eventually
> consistent with each other.

A failure in the registration process could have adverse consequences 
for the system: the aggregates could get out of synchronization with 
each other which may cause unpredicatable behavior in the system; some 
processes might end up as zombie processes continuing to run and use 
resources but never completing. The team identified the following 
specific failure scenarios related to the **RegistrationProcess** 
workflow. The workflow could: 

* Crash or be unable to persist its state after it receives an event but
  before it sends any commands. The message processor may not be able to
  mark the event as complete, so after a timeout the event is placed
  back in the topic subscription and re-processed.
* Crash after it persists its state but before it sends any commands.
  This puts the system into an inconsistent state because the workflow
  saved its new state without sending out the expected commands. The
  original event is put back in the topic subscription and 
  re-processed.
* Fail to mark that an event has been processed. The workflow will
  process the event a second time because after a timeout, the system
  will put the event back onto the service bus topic subscription.
* Timeout while it waits for a specific event that it is expecting. The
  workflow cannot continue processing and reach an expected end-state.
* Receive an event that it does not expect to receive while the workflow
  is in a particular state. This may indicate a problem elsewhere that
  implies that it is unsafe for the workflow to continue.
  
These scenarios can be summarized to identify two issues to address:

1. The **RegistrationProcess** handles an event successfully but fails
   to mark the message as complete. The **RegistrationProcess** will
   then process the event again after it is automatically returned to
   the subscription to the Windows Azure Service Bus topic.
2. The **RegistrationProcess** handles an event successfully, marks it
   as complete, but then fails to send out the commands.

### Making the System Resilient Whan an Event is Reprocessed

If the behavior of the workflow itself is idempotent, then if it 
receives and processes an event a second time then this does not result 
in any inconsistencies within the system. This approach would handle the 
first three failure conditions. After a crash, you can restart the 
workflow and reprocess the incoming event a second time. 

Instead of making the workflow idempotent, you could ensure that all the 
commands that the workflow sends are idempotent. Restarting the workflow 
may result in sending commands a second time, but if those commands are 
idempotent this will have no adverse affect on the process or the 
system. For this approach to work, you will still need to modify the 
workflow to guarantee that it sends all commands at least once. If the 
commands are idempotent, it doesn't matter if they are sent multiple 
times, but it does matter if a command is never sent at all. 

In the V1 release, most message handling is already either idempotent, 
or the system detects duplicate messages and sends them to a dead-letter 
queue. The exceptions are the **OrderPlaced** event and the 
**SeatsReserved** event. 

### Ensuring That Commands are Always Sent

To ensure that the system always sends commands when the 
**RegistrationProcess** workflow saves its state requires transactional 
behavior. This requires the team to implement a psuedo-transaction 
because it is not possible to enlist the Windows Azure Service Bus in a 
distributed transcation. 

The solution adopted by the team for the V3 release ensures that the 
system persists any commands that the **RegistrationProcess** tries to 
send but that fail. When the system next reloads the 
**RegistrationProcess** workflow, it tries to re-send the failed 
commands. If this fails, then the workflow cannot be loaded and cannot 
process any further messages until the cause of the failure is resolved. 

## Optimizing the Interactions Between the UI and the Domain

When a registrant creates an order, she visits the following sequence of 
screens in the UI. 

1. The Register screen. This screen displays the ticket types for the
   conference, the number of seats currently available. The registrant
   selects the quantities of each seat type that she would like to
   purchase.
2. The Registrant screen. This screen displays a summary of the order
   that includes a total price and a countdown timer that tells the
   registrant how long the seats will remain reserved. The registrant
   enters her details and preferred payment method.
3. The Payment screen. This simulates a third-party payment processor.
4. The Registration success screen. This displays if the payment
   succeeded. It displays to the registrant an order locator code and
   link to a screen that enables the registrant to assign attendees to
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

### Options to Reduce the Delay

The team discussed with the domain expert whether or not is always 
necessary to validate the seats availability before the UI sends the 
**RegisterToConference** conference command to the domain. 

> **BharathPersona:** This scenario illustrates some practical issues in
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

Most of the time, there are plenty of seats avaialable for a conference 
and registrants are not competing for the last few that are available. 
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

## Optimizing Command Processing 

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

> **BharathPersona:** It's also possible in the future that for some
> bounded contexts, the read-model will be hosted in a separate role
> instance from the write-model. Windows Azure Service Bus will
> transport the events that the system uses to construct the
> denormalized read-model.

There are a number of factors that the team will consider when they 
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

## Scaling Out

A further optimization that the team considered was to scale out the 
view model generators that populate the various read-models in the 
system. Every web-role that hosts a view model generator instance must 
handle the events published by the write-side by creating a subscription 
the the Windows Azure Service Bus topics. 

# Implementation Details 

This section describes some of the significant implementation details in 
this stage of the journey. You may find it useful to have a copy of the 
code so you can follow along. You can download a copy of the code from 
the repository on github: [mspnp/cqrs-journey-code][repourl]. 

> **Note:** Do not expect the code samples to exactly match the code in
> the reference implementation. This chapter describes a step in the
> CQRS journey, the implementation may well change as we learn more and
> refactor the code.

## Hardening the RegistrationProcess Workflow

**Note:** In the V2 release, the RegistrationProcess workflow generated a Guid in its constructor to use as an Id. This was removed as part of the hardening process in order to...



## Making the SeatsReserved Event Idempotent

https://github.com/mspnp/cqrs-journey-code/issues/475

## Creating a Psuedo Transaction when the Workflow Saves Its State and Sends a Command

It is not possible to have a transaction in Windows Azure that spans 
persisting the **RegistrationProcess** to storage and sending the 
command. Therefore the team decided to save any failed commands that the 
workflow sends, and to automatically retry those commands the next time 
that the system loads the workflow from storage. 

The following code sample from the **SqlProcessDataContext** class shows 
how the system persists the failed commands along with the state of the 
workflow: 

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
the workflow: 

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
https://github.com/mspnp/cqrs-journey-code/issues/473

# Testing 

During this stage of the journey the team re-organized the 
**Conference.Specflow** project in the **Conference.AcceptanceTests** 
Visual Studio solution to better reflect the purpose of the tests. 

## Integration Tests

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

## User Interface Tests

The **UserInterface** folder contains the acceptance tests. These tests 
are described in more detail in [Chapter 4, Extending and Enhancing the 
Orders and Registrations Bounded Context][j_chapter4]. The 
**Controllers** folder contains the tests that use the MVC controllers 
as the point of entry, and the **Views** folder contains the tests that 
use [WatiN][watin] to drive the system through its UI. 

[j_chapter4]:        Journey_04_ExtendingEnhancing.markdown

[repourl]:           https://github.com/mspnp/cqrs-journey-code
[watin]:             http://watin.org