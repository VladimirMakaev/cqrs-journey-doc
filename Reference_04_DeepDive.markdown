## Chapter 4

# A CQRS/ES Deep Dive (Chapter Title)

# The Domain Model  

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Aggregate Roots, Bounded Contexts etc.
  </span>
</div> 

# Commands and CommandHandlers 

## Commands

*Commands* are imperatives; they are requests for the system to 
perform a task or action. For example, "book two places on conference X" 
or "allocate speaker Y to room Z." Commands are usually processed just 
once, by a single recipient.

> I think that in MOST circumstances (if not all), the command should 
> succeed (and that makes the async story WAY easier and practical). You 
> can validate against the read model before submitting a command, and 
> this way being almost certain that it will succeed.  
> - Julian Dominguez (CQRS Advisors Mail List)

> When a user issues a Command, it'll give the best user experience if it 
> rarely fails. However, from an architectural/implementation point of 
> view, Commands will fail once in a while, and the application should be 
> able to handle that.  
> - Mark Seeman (CQRS Advisors Mail List)

## Command Handlers

> I don’t see the reason to retry the command here. When you see that 
> a command could not always be fulfilled due to race conditions, 
> go talk with your business expert and analyze what happens in this 
> case. How to handle compensation, offer an alternate solution, or deal 
> with overbooking. The only reason to retry I see is for technical 
> transient failures, like accessing the state storage. 
> - J&eacute;r&eacute;mie Chassaing (CQRS Advisors Mail List)

### Command Handlers without Event Sourcing

### Command Handlers with Event Sourcing

# Events and EventHandlers 

## Events and Intent

As previously mentioned events in event sourcing should capture the business intent, in addition to the change in state of the aggregate. The concept of intent is hard to pin down, as shown in the following conversation:

> *Developer #1*: One of the claims that I often hear for using event sourcing is that it enables
> you to capture the user's intent, and that this is valuable data. It may not be valuable right now, but
> if we capture it, it may turn out to have business value at some point in the future.

> *Developer #2*: Sure. For example, rather than saving a just a customer's latest address, we might
> want to store a history of the addresses the customer has had in the past. It may also be useful to know
> why a customer's address was changed: they moved house or you discovered a mistake with the existing address
> that you have on file.

> *Developer #1*: So in this example, the intent might help you to understand why the customer hadn't
> responded to offers that you sent, or might indicate that now might be a good time to contact the customer
> about a particular product. But isn't the information about intent, in the end, just data that you should
> store. If you do your analysis right, you'd capture the fact that the reason an address changes is an
> important piece of information to store?

> *Developer #2*: By storing events, we can automatically capture all intent. If we miss something
> during our analysis, but we have the event history, we can make use of that information later. If we capture 
> events we don't lose any potentially valuable data.

> *Developer #1*: But what if the event that you stored was just, "the customer address was changed"?
> That doesn't tell me why the address was changed.

> *Developer #2*: OK. You still need to make sure that you store useful events that capture what is
> meaningful from the perspective of the business.

> *Developer #1*: So what do events and event sourcing give me that I can't get with a well
> designed relational database that captures everything that I may need?

> *Developer #2*: It really simplifies things. The schema is simple. With a 
> relational database you have all the problems of versioning if you need to start storing new or
> different data. With an event sourcing, you just need to define a new event type. 

> *Developer #1*: So what do events and event sourcing give me that I can't get with a 
> standard database transaction log?

> *Developer #2*: Using events as your primary data model makes it very easy and natural to do time 
> related analysis of data in your system, for example:
> "what was the balance on the account at a particular point in time?" or, "what would the customer's
> status be if we'd introduced the reward program six months earlier?" The transactional data is not
> hidden away and inaccessible on a tape somewhere, it's there in your system.

> *Developer #1*: So back to this idea of intent. Is it something special that you can
> capture using events, or is it just some additional data that you save?

> *Developer #2*: I guess in the end, the intent is really there in the commands that orginate
> from the users of the system. The events record the consequences of those commands. If those events
> record the consequences in business terms then it makes it easier for you to infer the original intent of
> user.

> Thanks to Clemens Vasters and Adam Dymitruk

### How to Model Intent

This section examines two alternatives for modeling intent with reference to SOAP and REST style interfaces to help highlight the differences.

> **Note:** We are using SOAP and REST here as an analogy to help explain the differences between the approaches.

The following two code samples illustrate two, slightly different approaches to modeling intent alongside the event data:

**Example 1. The Event log or SOAP-style approach.**
```
[ 
  { "reserved" : { "seatType" : "FullConference", "quantity" : "5" }},
  { "reserved" : { "seatType" : "WorkshopA", "quantity" : "3" }},
  { "purchased" : { "seatType" : "FullConference", "quantity" : "5" }},
  { "expired" : { "seatType" : "WorkshopA", "quantity" : "3" }},
]
```

**Example 2. The Transaction log or REST-style approach.**

```
[ 
  { "insert" : { "resource" : "reservations", "seatType" : "FullConference", "quantity" : "5" }},
  { "insert" : { "resource" : "reservations", "seatType" : "WorkshopA", "quantity" : "3" }},
  { "insert" : { "resource" : "orders", "seatType" : "FullConference", "quantity" : "5" }},
  { "delete" : { "resource" : "reservations", "seatType" : "WorkshopA", "quantity" : "3" }},
]
```

The first approach uses an action-based contract that couples the events to a particular aggregate type. The second approach uses a uniform contract, that uses a **resource** field as a hint to associate the event with an aggregate type.

> **Note:** How the events are actually stored is a separate issue. This discussion is focusing on how to model your events.

The advantages of the first approach are:

* Strong typing.
* More expressive code.
* Better testability.

The advantages of the second approach are:

* Simplicity and a generic approach.
* Makes it easier to use existing internet infrastructure.
* Easier to use with dynamic languages and with changing schamas.

> **DeveloperPersona:** Variable environment state needs to be stored alongside events in order to have an accurate representation of the circumstances at the time when the command resulting in the event was executed, which means that we need to save everything!

# Embracing Eventual Consistency 

Maintaining the consistency of business data is a key requirement in all 
enterprise systems. One of the first things that many developers learn 
in relation to database systems is the ACID properties of transactions: 
transactions must ensure that the stored data is consistent as well 
transactions being atomic, isolated, and durable. Developers also become 
familiar with complex concepts such as pessimistic and optimistic 
concurrency, and their performance characteristics in particular 
scenarios. They may also need to understand the different isolation 
levels of transactions: serializable, repeatable reads, read committed, 
and read uncommitted. 

In a distributed computer system, there are some additional factors that 
are relevant to consistency. The CAP theorem states that it is 
impossible for a distributed computer system to provide the following 
three guarantees simultaneously: 

1. Consistency. A guarantee that all the nodes in the system see the 
   same data at the same time. 
2. Availability. A guarantee that the system can continue to operate
   even if a node is unavailable. 
3. Partition tolerance. A guarantee that the system continues to operate
   despite the nodes being unable to communicate. 

For more information about the [CAP theorem][captheorem], see CAP 
theorem on Wikipedia. 

The concept of *eventual consistency* offers to way to make these three 
guarantees simultaneously by relaxing the guarantee of consistency. In 
the CAP theorem, the consistency guarantee specifies that all the nodes 
should see the same data *at the same time*; if instead, we state that 
all the nodes will eventually see the same data, we can make the 
consistency guarantee compatible with the other two. Another way of 
viewing this is to say that we will accept that at any given time, some 
of the data seen by users of the system could be stale. For many 
business scenarios, this turns out to be perfectly acceptable: a 
business user will accept that the information they are seeing on a 
screen may be a few seconds, or even minutes out of date. Depending on 
the details of the scenario, the business user can refresh the display a 
bit later on to see what has changed, or simply accepts that what they 
see is always slightly out of date. There are some scenarios where this 
delay is unacceptable, but they tend to be the exception rather than the 
rule. 

> Very often people attempting to introduce eventual consistency into a 
> system run into problems from the business side. A very large part of 
> the reason of this is that they use the word consistent or consistency 
> when talking with domain experts / business stakeholders. 
> ...
> Business users hear 'Consistency' and they tend to think it means that 
> the data will be wrong. That the data will be incoherent and 
> contradictory. This is not actually the case. Instead try using the 
> word 'stale' or 'old', in discussions when the word stale is used the 
> business people tend to realize that it just means that someone could 
> have changed the data, that they may not have the latest copy of it.  
> Greg Young: [Quick Thoughts on Eventual Consistency][youngeventual] 



# Eventual Consistency and CQRS 

How does the concept of eventual consistency relate to the CQRS pattern? 
A typical implementation of the CQRS pattern is a distributed system 
made up of one node for the write-side, and one or more nodes for the 
read-side. Your implementation must provide some mechanism for 
synchronizing data between these two sides. This is not a complex 
synchronization task because all of the changes take place on the 
write-side, so the synchronization process only needs to push changes 
from the write-side to the read-side. 

If you decide that the two sides must always be consistent, then you 
will need to introduce a distributed transaction that spans both sides 
as shown in figure 1. 

![Figure 1][fig1]

**Using a distributed transaction to maintain consistency**

The problems that may result from this approach relate to performance 
and availability. Firstly, both sides will need to hold locks until both 
sides are ready to commit, in other words the transaction can only 
complete as fast as the slowest participant can. 

This transaction may include more than two participants. If we are 
scaling the read-side by adding multiple instances, the transaction must 
span all of those instances. 

Secondly, if one node fails for any reason or does not complete the 
transaction, the transaction cannot complete. In terms of the CAP 
theorem, by guaranteeing consistency, we cannot guarantee the 
availability of the system. 

If you decide to relax your consistency constraint and specify that your 
read-side only needs to be eventually consistent with the write-side, 
you can change the scope of your transaction. Figure 2 shows how you can 
make the read-side eventually consistent with the write-side by using a 
reliable messaging transport to propagate the changes. 

![Figure 2][fig2]

**Using a reliable message transport**

In this example, you can see that there is still a transaction. The 
scope of this transaction includes saving the changes to the data store 
on the write-side, and placing a copy of the change onto the queue that 
pushes the change to the read-side. 

This solution does not suffer from the potential performance problems 
that you saw in the original solution if you assume that the messaging 
infrastructure allows you to quickly add messages to a queue. This 
solution is also no longer dependent on all of the read-side nodes being 
constantly available because the queue acts as a buffer for the messages 
addressed to the read-side nodes. 

> **Note:** In practice, the messaging infrastructure is likely to use a 
  publish/subscribe topology rather than a queue to enable multiple 
  read-side nodes to receive the messages. 

This third example in figure z shows a way that you can do away with the 
need for a distributed transaction. 

![Figure 3][fig3]

**No distributed transactions**

This example depends on functionality in the write-side data store: it 
must be able to send a message in response to every update that the 
write-side model makes to the data. This approach lends itself 
particularly well to the scenario where you combine CQRS with event 
sourcing. If the event store can send a copy of every event that it 
saves onto a message queue, then you can make the read-side eventually 
consistent by using this infrastructure feature. 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	[Add a reference to an event store that offers this functionality.]
  </span>
</div>  

Queries 

# Optimizing the Read Side 

# Optimizing the Write Side 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	parallelizing command handling, events, etc.<br/>
	Snapshots in the event store
  </span>
</div>

## Concurrency and Aggregates

A simple implemention of aggregates and command handlers will load an aggregate instance into memory for each command that the aggregate must process. For aggregates that must process a large number of commands, you may decide to cache the aggregate instance in memory to avoid the need to reload it for every command.

If your system only has a single instance of an aggregate loaded into memory, that aggregate may need to process commands that are sent from multiple clients. By arranging for the system to deliver commands to the aggregate instance through a queue, you can ensure that the aggregate processes the commands sequentially. Also, there is no requirement to make the aggregate thread-safe, because it will only process a single command at a time.

In scenarios with an even higher throughput of commands, you may need to have multiple instances of the aggreagate loaded into memory, possibly in different processes. To handle the concurrency issues here, you can use event sourcing and versioning. Each aggregate instance must have a version number that is updated whenever the instance persists an event.
There are two ways to make use of the version number in the aggregate instance:

* **Optimistic:** Append the event to the event-stream if the the latest event in the event-stream is the same version as the current, in-memory, instance.

* **Pessimistic:** Load all the events from the event stream that have a version number greater than the version of the current, in-memory, instance.

> "These are technical performance optimizations that can be implemented on case-by-case basis."  
> Rinat Abdullin (CQRS Advisors Mail List)

# Messaging 

## Messaging and CQRS 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Discuss role of messaging infrastructure and CQRS. Transport for commands and events. Topologies: queues and pub/sub.
  </span>
</div> 

## Messaging Considerations 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Problems that arise with out of order messages, duplicate messages, lost messages. Strategies for dealing with these issues. 
	Need to discuss issues of dropped, multiple, out of order delivery here. <br />
	Multiple delivery of messages or lost messages. This is a general messaging issueIt is possible that due to a problem in the messaging infrastructureIf it is possible that the read-side could receive multiple copies of a single event, for example because of a problem in the messaging infrastructure, you must ensure that your system state remains correct. For example, an individual subscriber could receive multiple copies introducing a temporary error that you could fix by replaying the events. For example, if a reservation event is delivered twice, then you may end up with an inaccurate figure for the total number of bookings for a conference. Designing event messages to be idempotent or assigning messages unique identifiers are possible strategies to help address this problem. Similar problems can result from subscribers not receiving messages. <br />
	Out of order delivery. In some scenarios, the specific ordering of a set of event messages may be significant. Again, it may be that your messaging infrastructure does not guarantee ordered delivery. Assigning sequence numbers to messages, using batches, or using sagas are all possible strategies for addressing this problem. The chapter "A CQRS/ES Deep Dive" includes a discussion of sagas. <br />
  </span>
</div> 


## Messaging Infrastructure 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	What to expect from an infrastructure messaging service. <br />
	Reference Windows Azure Service Bus section below, but mention other products. 
  </span>
</div>

# Task-based UIs 

# Taking Advantage of Windows Azure

In the chapter "[Introducing the Command Query Responsibility 
Segregation Pattern][r_chapter2]," we suggested that the motivations for 
hosting an application in the cloud were similar to the motivations for 
implementing the CQRS pattern: scalability, elasticity, and agility. 
This section describes in more detail how a CQRS implementation might 
use some of specific features of the Windows Azure platform to provide 
some of the infrastructure that you typically need when you implement 
the CQRS pattern. 

## Scaling Out Using Multiple Role Instances 

When you deploy an application to Windows Azure, you deploy the 
application to roles in your Windows Azure environment; a Windows Azure 
application typically consists of multiple roles. Each role has 
different code and performs a different function within the application. 
In CQRS terms, you might have one role for the implementation of the 
write-side model, one role for the implementation of the read-side 
model, and another role for the UI elements of the application. 

After you deploy the roles that make up your application to Windows 
Azure, you can specify (and change dynamically) the number of running 
instances of each role. By adjusting the number of running instances of 
each role, you can elastically scale your application in response to 
changes in levels of activity. One of the motivations for using the CQRS 
pattern is the ability to scale the read-side and the write-side 
independently given their typically different usage patterns. For 
information about how to automatically scale roles in Windows Azure, see 
"[The Autoscaling Application Block][aab]" on MSDN. 

## Implementing an Event Store Using Windows Azure Table Storage 

## Implementing a Messaging Infrastructure Using the Windows Azure Service Bus 


[r_chapter1]:     Reference_01_CQRSContext.markdown
[r_chapter2]:     Reference_02_CQRSIntroduction.markdown
[r_chapter4]:     Reference_04_DeepDive.markdown


[captheorem]:	  http://en.wikipedia.org/wiki/CAP_theorem
[aab]:			  http://msdn.microsoft.com/en-us/library/hh680892(PandP.50).aspx
[youngeventual]:  http://codebetter.com/gregyoung/2010/04/14/quick-thoughts-on-eventual-consistency/

[fig1]:           images/Reference_04_Consistency_01.png?raw=true
[fig2]:           images/Reference_04_Consistency_02.png?raw=true
[fig3]:           images/Reference_04_Consistency_03.png?raw=true