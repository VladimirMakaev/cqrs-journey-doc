## Chapter 7
# Adding Resilience, New Bounded Contexts, and Features 

*To be decided*

# Adding Resilience and Adding Features

The top-level goal for this stage in the journey ...

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

During this stage of the journey... 

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

# Implementation Details 

This section describes some of the significant implementation details in 
this stage of the journey. You may find it useful to have a copy of the 
code so you can follow along. You can download a copy of the code from 
the repository on github: [mspnp/cqrs-journey-code][repourl]. 

> **Note:** Do not expect the code samples to exactly match the code in
> the reference implementation. This chapter describes a step in the
> CQRS journey, the implementation may well change as we learn more and
> refactor the code.

# Testing 

During this stage of the journey...


[repourl]:           https://github.com/mspnp/cqrs-journey-code
