## Chapter 4

# A CQRS/ES Deep Dive (Chapter Title)

# Introduction

This chapter begins with a brief recap of some of the key points from 
the previous chapters before exploring in more detail the key concepts 
related to CQRS and Event Sourcing (ES). 

## Read-models and Write-models

The CQRS pattern assigns the responsibility for modifying your 
application data and querying your application data to different sets of 
objects: a write-model and a read-model. The immediate benefit of this 
segregation is to clarify and simplify your code by applying the 
single-responsibilty principle: objects are responsible either for 
modifying data or querying data. 

However, the most important benefit of this segregation is that it is an 
enabler for making other changes to your application. 

## Commands and Data Transfer Objects

A typical approach to enabling a user to edit data is to use DTOs: the 
UI retrieves the data to be edited from the application as a DTO, a user 
edits the DTO in the UI, the UI sends the modified DTO back to the 
application, the application then applies those changes to the data in 
the database. 

This approach is data-centric and tends to use standard CRUD operations 
throughout. In the UI, the user performs operations that are essentially
CRUD operations.

This is a simple, well understood aproach that works well for many 
applications. However, for some applications it is more useful if the UI 
sends commands instead of DTOs back to the application to make changes 
to the data. Commands are behavior-centric instead of data-centric, 
maybe more intuitive to users, and can capture the user's intent more
effectively than DTOs. 

## Domain-driven Design (DDD) and Aggregates

Using commands enables you build a UI that is more closely aligned with 
the behaviors associated with your domain. Related to this are the DDD 
concepts associated with a rich domain model, focusing on aggregates as 
way to model consistency boundaries based on domain concepts. 

One of the advatages of using commands and aggregates instead of DTOs is 
to simplify locking and concurrency management in your application. 

## Data and Normalization

One of the changes that the CQRS pattern enables in your application is 
to segregrate your data as well as your objects. The write-model can use 
a database that is optimized for writes by being fully normalized. The 
read-model can use a database that is optimized for reads by being 
de-normalized to suit the specific queries that the application must 
support on the read-side. 

Several benefits flow from this: better performance because each 
database is optimized for a particlar set of operations, better 
scalability because you can scale-out each side independently, and 
simpler locking schemes. On the write side you no longer need to worry 
about how your locks impact queries, and on the read-side your database 
can be read-only. 

## Events and Event Sourcing

If you use databases on both the read-side and write-side you will still 
be performing CRUD operations on the database tables on the write-side 
and you will need a mechanism to push the changes from your normalized 
tables on the write-side to your de-normalized tables on the read-side. 

If you capture changes in your write-model as events, your can save all 
of your changes simply by appending those events to your database or 
data store on the write-side using only **Insert** operations. 

You can also use those same events to push your changes to the 
read-side. You can use those events to build projections that contain 
the data required by the queries on the read-side. 

## Eventual Consistency

If you use a single database in your application, your locking scheme 
determines what version of a record is returned by a query. This can be 
very complex if a query joins records from multiple tables. 

> **MarkusPersona:** Think about the complexities of how transaction
> isolation levels determine the locking behavior in a database.

Additionally, in a web application you have to consider that as soon as 
data is rendered in the UI it is potentially out of date because some 
other process or user could change it. 

If you segregate your data into a write-side store and a read-side 
store, you are now making it explicit in your architecture that when you 
query data it may be out of date, but that the data on the read-side 
will be *eventually consistent* with the data on the write-side. This 
helps you to simplify the design of the application and makes it easier 
to implement collaborative applications where multiple users may be 
trying to modify the same data simultaneously on the write-side. 

# The Domain Model  

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Aggregate Roots, Bounded Contexts etc.
  </span>
</div> 

# Commands and CommandHandlers 

This section describes the role of commands and command handlers in a 
CQRS implementation and shows an outline of how they might be 
implemented in the C# language. 

## Commands

Commands are imperatives; they are requests for the system to 
perform a task or action. For example, "book two places on conference X" 
or "allocate speaker Y to room Z." Commands are usually processed just 
once, by a single recipient.

Both the sender and the receiver of a Command should be in the same 
bounded context. You should not send a Command to another bounded 
context because you would be instructing that other bounded context, 
which has separate responsibilities in another consistency boundary, to 
perform some work for you. 

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

### Example Code

The following code sample shows a command and the **ICommand** interface 
that it implements. Notice that a command is a simple *Data Transfer 
Object* and that every instance of a Command has a unique Id. 

```Cs
using System;
	
public interface ICommand
{
	/// <summary>
	/// Gets the command identifier.
	/// </summary>
	Guid Id { get; }
}

public class MakeSeatReservation : ICommand
{
	public MakeSeatReservation()
	{
		this.Id = Guid.NewGuid();
	}

	public Guid Id { get; set; }

	public Guid ConferenceId { get; set; }
	public Guid ReservationId { get; set; }
	public int NumberOfSeats { get; set; }
}
```

## Command Handlers

Commands are sent to a specific recipient, typically an aggregate 
instance. The Command Handler performs the following tasks: 

1. It receives a Command instance from the messaging infrastructure.
2. It validates that the Command is a valid Command.
3. It locates the aggregate instance that is the target of the Command.
   This may involve creating a new aggregate instance or locating an
   existing instance.
4. It invokes the appropriate method on the aggregate instance passing
   in any parameters from the command.
5. It persists the new state of the aggregate to storage.

> I don’t see the reason to retry the command here. When you see that 
> a command could not always be fulfilled due to race conditions, 
> go talk with your business expert and analyze what happens in this 
> case. How to handle compensation, offer an alternate solution, or deal 
> with overbooking. The only reason to retry I see is for technical 
> transient failures, like accessing the state storage. 
> - J&eacute;r&eacute;mie Chassaing (CQRS Advisors Mail List)

Typically, you will organize your command handlers so that you have a 
class that contains all of the handlers for a specific aggregate type. 
The following sections show example command handler implementations for 
bounded contexts that don't use event sourcing and for bounded contexts 
that do use event sourcing. 

You messaging infrastructure should ensure that it delivers just a 
single copy of a command to single command handler. Commands should be 
processed once, by a single recipient. 

### Command Handlers without Event Sourcing

The following code sample shows a command handler class that handles 
commands for **Order** instances in a bounded context that does not use 
event sourcing. 

```Cs
using System;
using System.Linq;
using Common;
using Registration.Commands;

public class OrderCommandHandler :
	ICommandHandler<RegisterToConference>,
	ICommandHandler<MarkOrderAsBooked>,
	ICommandHandler<RejectOrder>,
	ICommandHandler<AssignRegistrantDetails>
{
	private Func<IRepository> repositoryFactory;

	public OrderCommandHandler(Func<IRepository> repositoryFactory)
	{
		this.repositoryFactory = repositoryFactory;
	}

	public void Handle(RegisterToConference command)
	{
		var repository = this.repositoryFactory();

		using (repository as IDisposable)
		{
			var tickets = command.Seats.Select(t => new OrderItem(t.SeatTypeId, t.Quantity)).ToList();

			var order = new Order(command.OrderId, command.ConferenceId, tickets);

			repository.Save(order);
		}
	}

	public void Handle(MarkOrderAsBooked command)
	{
		var repository = this.repositoryFactory();

            using (repository as IDisposable)
            {
                var order = repository.Find<Order>(command.OrderId);

                if (order != null)
                {
                    order.MarkAsBooked(command.Expiration);
                    repository.Save(order);
                }
            }
	}

	public void Handle(RejectOrder command)
	{
		...
	}

	public void Handle(AssignRegistrantDetails command)
	{
		...
	}
}
```

This handler handles four different commands for the **Order** 
aggregate. The **RegisterToConference** command is an example of a 
command that creates a new aggregate instance. The **MarkOrderAsBooked** 
command is an example of a command that locates an existing aggregate 
instance. Both examples use the **Save** method to persist the instance. 

> **Note:** If the aggregate generated any events when it processed the
> command, then these events are published when the repository saves the
> aggregate instance.

### Command Handlers with Event Sourcing

# Events and EventHandlers 

## Events and Intent

As previously mentioned events in event sourcing should capture the 
business intent, in addition to the change in state of the aggregate. 
The concept of intent is hard to pin down, as shown in the following 
conversation: 

> *Developer #1*: One of the claims that I often hear for using event 
> sourcing is that it enables you to capture the user's intent, and that 
> this is valuable data. It may not be valuable right now, but if we 
> capture it, it may turn out to have business value at some point in 
> the future. 
> 
> *Developer #2*: Sure. For example, rather than saving a just a 
> customer's latest address, we might want to store a history of the 
> addresses the customer has had in the past. It may also be useful to 
> know why a customer's address was changed: they moved house or you 
> discovered a mistake with the existing address that you have on file. 
> 
> *Developer #1*: So in this example, the intent might help you to 
> understand why the customer hadn't responded to offers that you sent, 
> or might indicate that now might be a good time to contact the 
> customer about a particular product. But isn't the information about 
> intent, in the end, just data that you should store. If you do your 
> analysis right, you'd capture the fact that the reason an address 
> changes is an important piece of information to store? 
> 
> *Developer #2*: By storing events, we can automatically capture all 
> intent. If we miss something during our analysis, but we have the 
> event history, we can make use of that information later. If we 
> capture events we don't lose any potentially valuable data. 
> 
> *Developer #1*: But what if the event that you stored was just, "the 
> customer address was changed"? That doesn't tell me why the address 
> was changed. 
> 
> *Developer #2*: OK. You still need to make sure that you store useful 
> events that capture what is meaningful from the perspective of the 
> business. 
> 
> *Developer #1*: So what do events and event sourcing give me that I 
> can't get with a well designed relational database that captures 
> everything that I may need? 
> 
> *Developer #2*: It really simplifies things. The schema is simple. 
> With a relational database you have all the problems of versioning if 
> you need to start storing new or different data. With an event 
> sourcing, you just need to define a new event type. 
> 
> *Developer #1*: So what do events and event sourcing give me that I 
> can't get with a standard database transaction log? 
> 
> *Developer #2*: Using events as your primary data model makes it very 
> easy and natural to do time related analysis of data in your system, 
> for example: "what was the balance on the account at a particular 
> point in time?" or, "what would the customer's status be if we'd 
> introduced the reward program six months earlier?" The transactional 
> data is not hidden away and inaccessible on a tape somewhere, it's 
> there in your system. 
> 
> *Developer #1*: So back to this idea of intent. Is it something 
> special that you can capture using events, or is it just some 
> additional data that you save? 
> 
> *Developer #2*: I guess in the end, the intent is really there in the 
> commands that originate from the users of the system. The events 
> record the consequences of those commands. If those events record the 
> consequences in business terms then it makes it easier for you to 
> infer the original intent of user. 

> Thanks to Clemens Vasters and Adam Dymitruk

### How to Model Intent

This section examines two alternatives for modeling intent with 
reference to SOAP and REST style interfaces to help highlight the 
differences. 

> **Note:** We are using SOAP and REST here as an analogy to help 
explain the differences between the approaches. 

The following two code samples illustrate two, slightly different 
approaches to modeling intent alongside the event data: 

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

The first approach uses an action-based contract that couples the events 
to a particular aggregate type. The second approach uses a uniform 
contract, that uses a **resource** field as a hint to associate the 
event with an aggregate type. 

> **Note:** How the events are actually stored is a separate issue. This 
discussion is focusing on how to model your events. 

The advantages of the first approach are:

* Strong typing.
* More expressive code.
* Better testability.

The advantages of the second approach are:

* Simplicity and a generic approach.
* Makes it easier to use existing internet infrastructure.
* Easier to use with dynamic languages and with changing schemas.

> **MarkusPersona:** Variable environment state needs to be stored 
alongside events in order to have an accurate representation of the 
circumstances at the time when the command resulting in the event 
was executed, which means that we need to save everything! 

## Events

### Sample Code

## EventHandlers

### Sample Code

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

This section discusses a number of issues that relate to the 
implementation of the read-side of the CQRS pattern. 

# Optimizing the Write Side 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	parallelizing command handling, events, etc.<br/>
	Snapshots in the event store
  </span>
</div>

## Concurrency and Aggregates

A simple implementation of aggregates and command handlers will load an 
aggregate instance into memory for each command that the aggregate must 
process. For aggregates that must process a large number of commands, 
you may decide to cache the aggregate instance in memory to avoid the 
need to reload it for every command. 

If your system only has a single instance of an aggregate loaded into 
memory, that aggregate may need to process commands that are sent from 
multiple clients. By arranging for the system to deliver commands to the 
aggregate instance through a queue, you can ensure that the aggregate 
processes the commands sequentially. Also, there is no requirement to 
make the aggregate thread-safe, because it will only process a single 
command at a time. 

In scenarios with an even higher throughput of commands, you may need to 
have multiple instances of the aggregate loaded into memory, possibly in 
different processes. To handle the concurrency issues here, you can use 
event sourcing and versioning. Each aggregate instance must have a 
version number that is updated whenever the instance persists an event. 

There are two ways to make use of the version number in the aggregate 
instance: 

* **Optimistic:** Append the event to the event-stream if the the latest 
event in the event-stream is the same version as the current, in-memory, 
instance. 
* **Pessimistic:** Load all the events from the event stream that have a 
version number greater than the version of the current, in-memory, 
instance. 

> "These are technical performance optimizations that can be implemented 
> on case-by-case basis." 
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
