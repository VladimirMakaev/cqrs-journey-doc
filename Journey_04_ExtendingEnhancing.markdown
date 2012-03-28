## Chapter 4
# Extending and Enhancing the Orders and Registrations Bounded Contexts 

*Exploring further*

# A Description of the Orders and Reservations Bounded Context 

The Orders and Reservations Bounded Context was discribed in some detail in the previous chapter. This chapter describes some changes that the team made in this bounded context during the second stage of their CQRS journey.

The specific topics described in this chapter include:

* Improvements to the way that message correlation works with the **ReservationProcessWorkflow**.
* Implementing a record locator to enable a registrant to retrieve an order that was saved during a previous session.
* Adding a countdown timer to the UI to enable a registrant to track how much longer tey have to complete an order.  

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

### Saga

In this bounded context, a saga is a class that coordinates the behavior 
of the aggregates in the domain. A saga subscribes to the events that 
the aggregates raise, and then follow a simple set of rules to determine 
which command or commands to send. The saga does not contain any 
business logic, simply logic to determine the next command to send. The 
saga is implemented as a state machine, so when the saga responds to an 
event, it can change its internal state in addition to sending a new 
command. 

The saga in this bounded context can receive commands as well as 
subscribe to events. 

## User Stories 

What were the key user stories addressed in this chapter?  

## Architecture 

What are the key architectural features? Server-side, UI, multi-tier, cloud, etc. 

# Patterns and Concepts 

* What are the primary patterns or approaches that have been adopted for this bounded context? (CQRS, CQRS/ES, CRUD, ...) 

* What were the motivations for adopting these patterns or approaches for this bounded context? 

* What trade-offs did we evaluate? 

* What alternatives did we consider? 


# Implementation Details 

Describe significant features of the implementation with references to the code. Highlight any specific technologies (including any relevant trade-offs and alternatives). 

Provide significantly more detail for those BCs that use CQRS/ES. Significantly less detail for more "traditional" implementations such as CRUD. 

# Testing 

Describe any special considerations that relate to testing for this bounded context.  
