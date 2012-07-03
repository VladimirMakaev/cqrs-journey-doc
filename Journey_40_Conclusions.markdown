# Chapter 8: Epilogue: Lessons Learned 

_How good was our map? Where did we get to? What did we learn? Did we get lost?_

> "I have never been lost, but I will admit to being confused for several weeks," Daniel Boone

This chapter summarizes the findings from our journey. It highlights 
what we feel were the most significant things that we learnt along the 
way, suggests some things that we would do differently if we were 
embarking on the journey with our new found knowledge, and points out 
some future paths for the Contoso Conference Management System that we 
or you might decide to take. 

You should bear in mind that this summary reflects our specific journey 
and that not all of these findings will necessarily apply to your own 
CQRS journeys. For example, one of the goals of our journey was to 
explore how to implement the CQRS pattern in a application that is 
deployed to Windows Azure and that makes use of the scalability and 
reliability offered by the cloud. For our project, this meant using 
messaging to enable multiple role types and instances to communicate 
with each other. It may be that your project does not require multiple 
role instances or is not deployed to the cloud and therefore may not 
need to use messaging so extensively (or at all). 

We hope that these findings do prove useful, especially if you are just 
starting out with CQRS and event sourcing. 

# What Did we Learn?

This section describes our key learnings. They are not presented in any 
particular order. 

## Performance matters

At the start of our journey, one of our understandings about the CQRS 
pattern was that by separating the read and write sides of the 
application we could optimize each for performance. This perspective is 
shared by many in the CQRS community, for example: 

> "CQRS taught me that I can optimize reads and writes separately, it
> doesn't mean I have to denormalize into flat tables manually all the
> time."  
> Kelly Sommers

This was born out in practice during our journey and we benefited 
significantly from this separation when we did need to solve a 
performance issue. 

During the last stage of our journey our testing did reveal a set of 
performance issues in our application. When we investigated these, it 
turned out they had less to do with the way that we had implemented the 
CQRS pattern and more to do with the way that we were using our 
infrastructure. Discovering the root cause of these problems was the 
hard part: with so many moving parts in the application, getting the 
right tracing and the right data for analysis was the challenge. Once we 
identified the bottlenecks, fixing them turned out to be relatively 
easy, largely because of the way that the CQRS pattern enables you to 
clearly separate out different elements, such as reads and writes, in 
the system. Although the separation of concerns that results from 
implementing the CQRS pattern can make it harder to indentify an issue, 
once you have identified an issue it is not only easier to fix but also 
easier to keep fixed: the decoupled architecture means that it is easier 
to write unit tests that reproduce issues. 

The challenges we encountered in tackling the performance issues in the 
system were more to do with the fact that our system is a distributed, 
message-based system than the fact that it implements the CQRS pattern. 

Chapter 7, "[Adding Resilience and Optimizing Performance][j_chapter7]" 
provides more information about the ways that we addressed the 
performance issues in the system and makes some suggestions about additional changes that we would like to make, but didn't have time to implement. 

## Implementing a message-driven system is far from simple

Our approach to infrastructure on this project was to develop it as 
needed during the journey. We didn't anticipate (and had no forewarning 
of) how much time and effort we would need to create the robust 
infrastructure that our application required: we spent at least twice as 
much time as we originally planned on many development tasks because we 
continued to uncover additional infrastructure related requirements. In 
particular we learnt that having a robust event store from the begining 
is essential. Another key learning is that all IO on the message bus 
should be asynchronous.

> **JanaPersona:** Although our event store is not production-ready, the
> current implementation gives a good indication of the type of issues
> you should address if you decided to implement your own event store.

Although our application is large, it illustrated clearly to us the 
importance of having end-to-end tracing available, and the value of 
tools that help to understand all of the message flows in the system. 
Chater 4, "[Extending and Enhancing the Orders and Registrations Bounded 
Context][j_chapter4]" describes the value of tests in helping to 
understand the system, and the Messaging Intermediate Lanagues (MIL) 
created by Josh Elster, one of our advisors. 

> **GaryPersona:** It would also help if we had a standard notation for
> messaging that would help us communicate some of the issues with the
> domain experts and people outside of the core team.

In summary, many of the issues we met along the way in our journey were 
not related to specifically to the CQRS pattern, but were more related 
to the distributed, message-driven nature of our solution. 

## The cloud has challenges

Although the cloud brings many benefits such as reliable, scalable, off-the-shelf services that you can provision with just a few mouse clicks, cloud environments also introduce some challenges: 

* You may not be able to use transactions in all places you'd like to 
  because the distributed nature of the cloud makes ACID transactions
  impractical in many scenarios. Therefore, you need to understand how
  to work with eventual consistency. For examples, see chapter 5,
  "[Preparing for the V1 Release][j_chapter5]," and the section _Options
  to reduce the delay in the UI_ in chapter 7, "[Adding Resilience and
  Optimizing Performance][j_chapter7]."
* You may want to re-examine your assumptions about how to organize your
  application into different tiers. For example, see the discussion
  around in-process, synchronous commands in in chapter 7, "[Adding
  Resilience and Optimizing Performance][j_chapter7]."
* You must take into account not only the latency between the browser or
  on-premises environment and the cloud, but also the latency between
  the different parts of your system that are running in cloud.
* You must take into account transient errors and be aware of how
  different cloud services might implment throttling. If your application
  uses several cloud services that might be throttled, you must
  coordinate how your application handles being throttled by different
  services at different times.

> **MarkusPersona:** We found that having a single **bus** abstraction
> in our code obscurred the fact that some messages are handled locally
> in process and some are handled in a different role instance. To see
> how this is implemented, look at the **ICommandBus** interface and the
> **CommandBus** and **SynchronousCommandBusDecorator** classes. Chapter
> 7, "[Adding Resilience and Optimizing Performance][j_chapter7]
> includes a discussion of **SynchronousCommandBusDecorator** class.

A complex cloud environment can make it harder to run quick tests during 
development. A local test environment may not mimic the behavior of the 
cloud exactly, especially in relation to performance. 

> **Note:** The multiple build configurations in our Visual Studio
> solution were partially designed to address this, but also to help
> people downloading and playing with the code to get started quickly.

## CQRS is different

At the start of our journey we were warned that although the CQRS 
pattern appears to be simple, in practice it requires a significant 
shift in the way that you think about many aspects of the project. Again,
this was born out by our experiences during our journey. You must be 
prepared to throw away many assumptions and pre-conceived ideas and you 
will probably need to implement the CQRS pattern in several bounded 
contexts before you begin to fully understand the benefits you can 
derive from the pattern. 

An example of this is the concept of eventual consistency. If you come 
from a relational database background and the ACID properties of 
transactions, then embracing eventual consistency and understanding its 
implications at all levels in the system is a big step to take. Chapter 
5 "[Preparing for the V1 Release][j_chapter5]" and chapter 7 "[Adding 
Resilience and Optimizing Performance][j_chapter7]" both discuss 
eventual consistency in different areas of the system. 

In addition to being different from what you might be familiar with, 
there is also no single correct way to implement the CQRS pattern. We 
made more false starts on pieces of funcionality and estimated poorly 
how long things would take due to our unfamiliarity with the pattern and 
approach. As we become more comfortable with the approach, we hope we 
will become faster at identifiying how to implement the pattern in 
specific circumstances and improve the accuracy of our estimates. 

> **MarkusPersona:** The CQRS pattern is conceptually simple: the devil
> is in the details.

Another example where we took some time to understand the CQRS approach 
and its implications was in the integration between our bounded 
contexts. Chapter 5 "[Preparing for the V1 Release][j_chapter5]" 
includes a detailed discussion of how the team approached the 
integration issue between the Conference Management and the Orders and 
Registrations bounded contexts. This part of the journey uncovered some 
addtional complexity that relates to the level of coupling between 
bounded contexts when you use events as the integration mechanism. Our 
assumption that events should only contain information about the change 
in the aggregate or the bounded context proved to be unhelpful: events 
can contain additional information that is useful to the subscriber and 
helps to reduce the amount of work that a subscriber must perform. 

The CQRS pattern introduces additional considerations in how to 
partition your system. Not only do you need to consider how to partition 
your system into tiers, but you also need to consider how to partition 
your system into bounded contexts (some of which will contain 
implementations the CQRS pattern). We revised some of our assumptions about 
tiers in the last stage of our journey, bringing some processing into 
our web roles from the worker role where it was originally done. This 
is described in chapter 7 "[Adding Resilience and Optimizing 
Performance][j_chapter7]" in the section that discusses moving some 
command processing in-process. Partitioning the system into bounded 
contexts should be done based on your domain model: each bounded context 
has its own domain model and ubiquitous language. Once you have 
identified your bounded contexts, you can then identify in which bounded 
contexts to implement the CQRS pattern. This affects how and where you 
need to implement integration between these isolated bounded contexts. 
Chapter 2 "[Decomposing the Domain][j_chapter2]" introduces our 
decisions for the Contoso Conference Management System. 

**GaryPersona:** A single process (role instance in our deployment) can 
host multiple bounded contexts. In this scenario, you don't necessarily 
need to use messaging for the bounded contexts to communicate with each 
other. 

Implementing the CQRS pattern is more complex than implementing a 
traditional CRUD-style system. Our experiences during the journey have 
clearly confirmed to us why the CQRS pattern is not a top-level 
architecture. You must be sure that the costs associated with 
implementing a CQRS-based bounded context with this level of complexity 
are worth it; in general, it is in high-contention, collaborative 
domains that you will see the benefits of the CQRS pattern.

> **GaryPersona:** Analyzing the business requirements, building a
> useful model, maintaining the model, expressing 
> it in code, and implementing it using the CQRS pattern all take time
> and cost money. If this is the first time you have implemented the
> CQRS pattern, you'll also have the overhead of investing in your
> infrastructure elements such as message buses and event stores.

## Event sourcing and transaction logging

We had some debate about whether or not event sourcing and transaction 
logging amount to the same thing: they both create a record of what 
happened, and they both enable you to recreate the state of your system 
by replaying the historical data. The conclusion was that the 
distinguishing feature is that events capture intent in addition to 
recording the facts of of what happened. For more detail on what we mean 
by intent, see chapter 4, "[A CQRS and ES Deep Dive][r_chapter4]",in the 
reference guide. 

## Involving the domain expert

Implementing the CQRS pattern encourages involvement of the domain 
expert. The pattern enables you to separate out the domain on the 
write-side and the reporting requirements on the read-side and to 
separate these from infrastructure concerns. This separation makes it 
easier to involve the domain expert in those aspects of the system where 
his expertise is most valuable. The use of domain-driven design concepts 
such as bounded contexts and the ubiquitous language also help to focus 
the team and to foster clear communication with the domain expert. 

Our acceptance tests proved to be an effective way to involve the domain 
expert and capture his knowledge. Chapter 4 "[Extending and Enhancing 
the Orders and Registrations Bounded Context][j_chapter4]" describes 
this tesing approach in detail. 

> **JanaPersona:** As a side-effect, these acceptance tests also
> contributed to our ability to handle our pseudo-production releases
> quickly because they enables us to run a full set of tests at the UI
> level to verify the behavior of the system in addition to the unit and
> integration tests.

In addition to helping the team define the functional requirements of 
the system, the domain expert should also be involved in evaluating the 
trade-offs between consistency, availability, durability, and costs. For 
example, the domain expert should help to identify when a manual 
process is acceptable and what level of consistency is required in 
different areas of the system. 

> **GaryPersona:** Developers have a tendency to try and lock everything
> down to transactions guaranteeing full consistency: sometimes it's
> just not worth the effort.

## When to use CQRS

A the end of our journey, we can suggest some of the criteria you should 
evaluate to determine whether or not you should consisder implementing 
the CQRS pattern in one or more bounded contexts in your application. 
The more of these questions you can answer positively, the more likely 
it is that applying the CQRS pattern to a given bounded context will
bring net benefits to your solution: 

* Does the bounded context implement an area of
  business functionality that is a key differentiator in your market?
* Is the bounded context collaborative in nature with elements that are
  likely to have high-levels of contention at run time? By collaborative
  we mean multiple users competing for access to the same resources.
* Is the bounded context likely to experience ever-changing business
  rules?
* Do you have a robust, scalable messaging and persistence
  infrastructure already in place?
* Is scalability one of the challenges facing this bounded context?
* Is the business logic in the bounded context complex?
* Are you clear about the benefits that the CQRS pattern will bring to
  this bounded context?
  
**GaryPersona:** These are rules of thumb, not hard and fast rules.


# What would we do differently if we started over? 

This section is a result of our reflection on our journey and identifies 
some things that we'd do differently and some other opportunities that 
we'd like to pursue if we were starting over with the knowledge of the 
CQRS pattern and event sourcing that we now have. 

## Start with a solid infrastructure for messaging and persistence

We'd start with a solid messaging and persistence infrastructure. The 
approach we took, starting simple and building up the infrastructure as 
required meant that we built up technical debt during the journey. We 
also found that taking this approach meant that in some cases, the 
choices we made about the infrastructure affected the way that we 
implemented the domain. 

> **JanaPersona:** From the perspective of the journey, if we had
> started with a solid infrastructure, we would have had time to tackle
> some of the more complex parts of the domain such as wait-listing.

Starting with a solid infrastructure would also enable us to 
start performance testing earlier. We'd also do some more research 
into how other people do their performance testing on CQRS-based 
systems, and seek out performance benchmarks on other systems such as 
Jonathan Oliver's [EventStore][jolivereventstore]. 

One of the reasons we took the approach that we did was the advice we 
received from our advisors: "Don't worry about the infrastructure." 

## Leverage the capabilities of the infrastructure more

Starting with a solid infrastructure would also allow us to make more 
use of the capabilities of the infrastructure. For example, we use the 
identity of the message originator as the value of the session id in 
Windows Azure Service Bus when we publish an event: this is not always 
the best use of the session id from the perspective of the parts of the 
system that process the event. 

As part of this, we'd also investigate how the infrastructure could 
support other special cases of eventual consistency such as; timing 
consistency, monotonic consistency, "read my writes," and 
self-consistency. 

Another area we'd like to explore is using the infrastructure to support 
migration between versions: instead if treating migration in an ad-hoc 
way for each version we could investigate using a message-based or 
real-time communication process to coordinate bringing the new version 
online. 

## Adopt a more systematic approach to implementing process managers

We began to implement our process manager very early in the journey and 
were still hardening it and ensuring that its behavior was idempotent in 
the last stage of the journey. Again, starting with some solid 
infrastructure support for process managers to make them more resilient 
would have helped us. However, if starting over, we'd also wait to 
implement a process manager until a later stage in the journey rather 
than diving straight in.

We began implementing the **RegistrationProcessManager** class 
during the first stage in our journey. The intial implementation is 
described in Chapter 3, "[Orders and Registrations Bounded 
Context][j_chapter3]." We made changes to this process manager during 
every subsequent stage of our journey. 

## Partition the application differently

We'd think more carefully at the start of the project about the tiering 
of the system. We found that the way that we partitioned the application 
into web roles and worker roles as described in Chapter 4 "[Extending 
and Enhancing the Orders and Registrations Bounded Context][j_chapter4]" 
was not optimal, and in the last stage of the journey, Chapter 7 
"[Adding Resilience and Optimizing Performance][j_chapter7]" we made 
some major changes to this architecture as part of the performance 
optimization effort. 

As a part of this reorganization in the last stage of the journey we 
introduced synchronous command processing alongside the pre-existing 
asynchronous command processing. We would try to use either synchronous 
or asynchronous command processing in the system, not both. 

## Organize the development team differently

The approach we took to learning about the CQRS pattern was to iterate: 
develop, go back, discuss, and then refactor. From the learning 
perspective, we may have gained more by having several developers work 
independently on the same feature and then compare the result: this may 
have uncovered a broader set of solutions and approaches. 

## Evaluate how appropriate the domain and the bounded contexts are for the CQRS pattern

We'd like to start with a clearer set of heuristics, such as those 
outlined earlier in this chapter, to determine whether or not a 
particular bounded context will benefit from the CQRS pattern. We may 
have learnt more if we had focused on a more complex area of the domain 
such as wait-listing instead of on the Orders and Registrations bounded 
context and Payments bounded context. 

## Plan for performance

We'd address performance issues much earlier in the journey. In 
particular, we'd: 

* Set clear performance goals ahead of time.
* Run performance tests much earlier in the journey.
* Use larger and more realistic loads.

We didn't do any performance testing until the last stage of our 
journey. For a detailed discussion of the issues we found and how we 
addressed them, see chapter 7 "[Adding Resilience and Optimizing 
Performance][j_chapter7]." 

## Think about the UI differently

We felt that the way our UI interacts with the write models and read 
models, and handles eventual consistency worked well and met the 
requirements from the business. 

We'd like investigate other ways to avoid waiting in the UI unless its 
absolutely necessary, perhaps by using browser push techniques. The UI 
in the current system still needs to wait, in some places, for 
asynchronous updates to take place against the read-model. 

## Explore some of additional benefits of event sourcing

We found during the third stage of our journey, described in Chapter 5, 
"[Preparing for the V1 Release][j_chapter5]", that modifying the Orders 
and Registrations bounded context to use event sourcing helped to 
simplify the implementation of this bounded context, in part because it 
was already using a large number of events. 

In the current journey, we didn't get a chance to explore the further 
promises of flexibility and the ability to mine past events for new 
business insights from event sourcing. However, we did ensure that the 
system persists copies of all events (not just those that are needed to 
rebuild the state of the aggregates) and commands to enable these types 
of scenario in the future.

> **GaryPersona:** It would also be interesting to investigate whether
> the ability to mine past event streams for new business insights is
> easier to achieve with event sourcing or other technologies such as
> database transaction logs or the [StreamInsight][streaminsight]
> feature of SQL Server.

## Explore the issues associated with integrating bounded contexts 

In our V3 release, all of the bounded contexts are implemented by same 
core development team. We'd like to investigate how easy it is, in 
practice, to integrate a bounded context implemented by a different 
development team with the existing system. 

This is a great opportunity for you to contribute to the learning 
experience: go ahead and implement another bounded context (see the 
outstanding stories in the [product backlog][backlog]), integrate it 
into the Contoso Conference Management System, and write another chapter 
of the journey describing your experiences. 

[j_chapter2]:        Journey_02_BoundedContexts.markdown
[j_chapter3]:        Journey_03_OrdersBC.markdown
[j_chapter4]:        Journey_04_ExtendingEnhancing.markdown
[j_chapter5]:        Journey_05_PaymentsBC.markdown
[j_chapter6]:        Journey_06_V2Release.markdown
[j_chapter7]:        Journey_07_V3Release.markdown
[r_chapter4]:        Reference_04_DeepDive.markdown
[jolivereventstore]: https://github.com/joliver/EventStore
[backlog]:           https://github.com/mspnp/cqrs-journey-code/issues?labels=Type.Story%2CStat.Pending&page=1&state=open
[streaminsight]:     http://www.microsoft.com/sqlserver/en/us/solutions-technologies/business-intelligence/complex-event-processing.aspx

