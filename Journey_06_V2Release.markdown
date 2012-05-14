## Chapter 6
# Versioning our System 

*"Variety is the very spice of life". William Cowper,Olney Hymns (1779)*

# Upgrading the Code and Migrating the Data 

The top-level goal for this stage in the journey is to learn about how to upgrade a system that includes bounded contexts that implement the CQRS pattern and event sourcing. The user stories that the team implemented in this stage of the journey involve both changes to the code and changes to the data: some existing data schemas changed and new data schemas were added. In addition to upgrading the system and migrating the data, the team planned to do the upgrade and migration with no downtime for the live system running in Windows Azure.

## Working Definitions for this Chapter 

Outline any working definitions that were adopted for this chapter. 

## User Stories 

The team implemented the following user stories during this phase of the project.

### No Downtime Upgrade

The goal for the V2 release is to perform the upgrade, including any necessary data migration, without any downtime for the system. If this is not feasible with the current implementation, then the downtime should be minimized, and the system should be modified to support zero downtime upgrades in the future (starting with the V3 release).

> **BethPersona:** Ensuring that we can perform no downtime upgrades is
> crucial to our credibility in the marketplace.

### Display Remaining Seat Quantities

Currently, when a registrant creates an order, there is no indication of the number of seats remaining for each seat type. The UI should display this information when the registrant is selecting seats for purchase.

### Handle Zero-cost Seats

Currently, when a registrant selects seats that have a zero-cost, the UI flow still takes the registrant to the payments page even though there is nothing to pay. The system should detect when there is nothing to pay and adjust the flow to take the registrant directly to the conformation page for the order.

### Support for Discounts to Seat Prices

Registrants should be able to obtain discounts through the use of **Promotional Codes**. A registrant can enter a **Promotional Code** during the ordering process and the system will adjust the total cost of the order appropriately.

The discount associated with an individual **Promotional Code** can be from 1% to 100%. Each **Promotional Code** has an associated quota that determines how many seats are available at the dicounted price (this may be unlimited) and a scope that determines which seat type the code is associated with; this can also apply to all seat types rather than just one.

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
    Updated to include details of single-use codes and cummulative codes.
  </span>
</div>

## Architecture 

What are the key architectural features? Server-side, UI, multi-tier, cloud, etc. 

# Patterns and Concepts 

* What are the primary patterns or approaches that have been adopted for this bounded context? (CQRS, CQRS/ES, CRUD, ...) 

* What were the motivations for adopting these patterns or approaches for this bounded context? 

* What trade-offs did we evaluate? 

* What alternatives did we consider?

## Handling Changes to Events Definitions

When the team examined the requirements for the V2 release, it became 
clear that they would need to change some of the events used in the 
Orders and Registrations bounded context to accomodate some of the new 
features: the **RegistrationProcess** would change and the system would 
provide a better user experience when the order had a zero cost. 

The Orders and Registrations bounded context uses event sourcing, so 
after the migration to V2 then event store will contain the old events 
but will start saving the new events. When events are replayed, the 
system will need to operate correctly when it processes both the old and 
new sets of events. 

The team considered two approaches to handle this type of change in the 
system. 

### Mapping/Filtering Event Messages in the Infrastructure

This option handles old event messages and message formats by dealing 
with them somewhere in the infrastructure before they reach the domain. 
You can filter out old messages that are no longer relevant and use 
mapping to transform old format messages to a new format. This approach 
is initially the more complex approach because it requires changes in 
the infrastructure, but has the advantage of keeping the domain _pure_ 
because the domain only needs to understand the current set of events. 

### Handling Multiple Message Versions in the Aggregates

This alternative passes all the message types (both old and new) through 
to the domain where each aggregate must be able to handle both the old 
and new messages. This may be an appropriate strategy in the short-term, 
but will eventually cause the domain-model to become polluted with 
legacy event handlers. 

## Honor Message Idempotency

One of the key issues to address in the V2 release is to make the system 
more robust. In the V1 release, in some scenarios, it is possible that 
some messages might be processed more than once and result in incorrect 
or inconsistent data in the system. 

> **JanaPerson:** Message idempotency is important in any system that
> use messaging, not just systems that implement the CQRS pattern or use
> event sourcing.

In some scenarios, it would be possible to design idempotent messages, 
for example by using a message that says "set the seat quota to 500" 
rather than a message that says "add 100 to the seat quota." You could 
safely process the first message multiple times, but not the second. 

However, it is not always possible to use idempotent messages, so the 
team decided to use the de-duplication feature of the Windows Azure 
Service Bus to ensure that messages are only delivered once. The team 
made some changes to the infrastructure to ensure that Windows Azure 
Service Bus can detect duplicate messages, and configured Windows Azure 
Service Bus to perform duplicate message detection. 

To understand how this is implemented, see the section "De-duplicating 
Messages" below. 

Additionally, you need to consider how the message handlers in the 
system retrieve messages from queues and topics. The current approach 
uses the Windows Azure Service Bus peek/lock mechanism. This is a three 
stage process: 

1. The handler retrieves a message from the queue or topic and leaves a
   locked copy of the message on the queue or topic. Other clients
   cannot see or access locked messages.
2. The handler processes the message.
3. The handler deletes the locked message from the queue. If a locked
   message is not unlocked or deleted after a fixed time, the message is
   unlocked and made available so that it can be retrieved again.

If step 3 fails for some reason, this means that the system can process 
the message more than once.

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
    Need to add details of how this is resolved. See https://github.com/mspnp/cqrs-journey-code/issues/266
  </span>
</div>

## Persisting Integration Events

One of the concerns raised with the V1 release was about the way that 
the system persists the integration events that are sent from the 
Conference Management bounded context to the Orders and Registrations 
bounded context. These events include information about conference 
creation and publishing, and details of seat types and quota changes. 

In the V1 release, the **ConferenceViewModelGenerator** class in the 
Orders and Registrations bounded context handles these events by 
updating its view model and sending commands to the 
**SeatsAvailability** aggregate to tell it to change its seat quota 
values. 

This approach means that the Orders and Registrations bounded context is 
not storing any history and this could potentially cause problems: for 
example, other views look up seat type descriptions from this projection 
that only contains the latest value of the seat type description, as a 
result replaying a set of events elsewhere may regenerate another 
read-model projection that contains incorrect seat type descriptions. 

The team considered the following four options:

* Save all of the events in the originating bounded context (the
  Conference Management bounded context) and use a shared event store
  that the Orders and Registrations bounded context can access to replay
  these events. The receiving bounded context could replay the event
  stream up to a point in time when it needed to see what the seat type
  description was previously.
* Save all of the events as soon as they arrive in the receiving bounded
  context (the Orders and Registrations bounded context).
* Let the command handler in the view model generator save the events,
  selecting only those that it needs.
* Let the command handler in the view model generator save diferent
  events, in effect using event sourcing for this view model.

The first option is not always viable. In this particular case it would 
work because the same team is implementing both bounded contexts and the 
infrastructure making it easy to use a shared event store. 

> **BharathPersona:** Although from a purist's perspective, the first
> option breaks the strict isolation between bounded contexts, in some
> scenarios it may be an acceptable and pragmatic solution.

A possible risk with the third option is that the set of events that are 
needed may change in the future. If we don't save events now, they are 
lost for good. 

## Message Ordering

The acceptance tests that the team created and ran to verify the V1 
release highlighted a potential issue with message ordering: the 
acceptance tests that exercised the Conference Management bounded 
context sent a sequence of commands to the Orders and Registrations 
bounded context that sometimes arrived out of order. 

> **MarkusPersona:** This effect was not noticed when a human user
> tested this part of the system because the time delay between the
> times that the commands were sent was much greater making it less
> likely that the messages would arrive out of order.

The team considered two alternatives for ensuring messages are 
guaranteed to arrive in the correct order. 

* The first option is to use message sessions; a feature of the Windows
  Azure Service Bus. If you use message sessions, this offers guarantees
  that messages within a session are delivered in the same order that
  they were sent.
* The second alternative is to modify the handlers within the
  application to detect out of order messages through the use of
  sequence numbers or timestamps added to the messages when they are
  sent. If the receiving handler detects an out of order message, it
  rejects the message and puts it back onto the queue or topic to be
  processed later, after it has processed the messages that were sent
  before the rejected message.

The preferred solution in this case is to use Windows Azure Service Bus 
message sessions because this requires less change to the exsiting code. 
Both approaches would introduce some additional latency into the message 
delivery, but the team does not anticipate that this will have a 
significant effect on the performance of the system. 

## Command Optimizations

The current implementation uses the same messaging infrastructure for 
both commands and events. The Windows Azure Service Bus provides a 
reliable, performant, and scalable infrastructure for messaging in 
Windows Azure. The team plans to evaluate whether the same 
infrastructure is necesssary for both commands and events. 

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

There are two factors that the team will consider when they determine 
whether to continue using the Windows Azure Service Bus for transporting 
command messages. 

* In a CQRS implementation, commands are typically used within a bounded
  context, not between bounded contexts. This may mean that a command
  only exists within a process (or role instance in Windows Azure) and
  could therefore be handled in-memory without the need for any more
  robust messaging infrastructure.
* You can treat commands as a two-way, synchronous messaging pattern.

> An asynchronous command doesn't exist, it's actually another event. If
> I must accept what you send me and raise an event if I disagree, it's
> no longer you telling me to do something, it's you telling me
> something has been done. This seems like a slight difference at first,
> but it has many implications.  
> Greg Young - Why lot's of developers use one-way command messaging 
> (async handling) when it's not needed? - DDD/CQRS Google Groups

## Implement Per Handler Subscriptions

# Implementation Details 

Describe significant features of the implementation with references to the code. Highlight any specific technologies (including any relevant trade-offs and alternatives). 

Provide significantly more detail for those BCs that use CQRS/ES. Significantly less detail for more "traditional" implementations such as CRUD. 

## De-duplicating Messages

# Testing 

Describe any special considerations that relate to testing for this bounded context.  
