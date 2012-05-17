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

This is the option the team selected for the V2 release because it 
involved the minimum amount of code changes. 

> **JanaPersona:** Dealing with both old and new events in the
> aggregates now does not prevent you from later moving to the first
> option and using a mapping/filtering mechanism in the infrastructure.

## Honor Message Idempotency

One of the key issues to address in the V2 release is to make the system 
more robust. In the V1 release, in some scenarios, it is possible that 
some messages might be processed more than once and result in incorrect 
or inconsistent data in the system. 

> **JanaPersona:** Message idempotency is important in any system that
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

The purpose of persisting the events is to enable them to be played back 
when the the Orders and Registrations bounded context needs the 
information about current seat quotas in order to calculate the number 
of remaining seats. To calculate these numbers consistently, you must 
always play the events back in the same order. There are several choices 
for this ordering: 

* The order the events were sent by the Conference Management bounded
  context.
* The order the events were received by the Orders and Registrations
  bounded context.
* The order the events were processed by the Orders and Registrations
  bounded context.

Most of the time these orderings will be the same. There is no correct 
order, you just need to choose one to be consistent. Therefore, the 
choice is determined by simplicity: in this case the simplest approach 
is to persist the events in the order that the handler in the Orders and 
Registrations bounded context receives them. 

> **MarkusPersona:** This choice does not typically arise with event
> sourcing. Aggregates create events in a fixed order, and that is the
> order that the system uses to persist the events. In this scenario,
> the integration events are not created by a single aggregate.

There is a similar issue with saving timestamps for these events. 
Timestamps may be useful in the future if these is a requirement to look 
at number of remaining seats at a particular time. The choice here is 
whether you should create a timestamp when the event is created in the 
Conference Management bounded context or when it is received by the 
Orders and Registrations bounded context. It's possible that the Orders 
and Registrations bounded context is offline for some reason when the 
Conference Management bounded context creates an event, therefore the 
team decided to create the timestamp when the Conference Management 
bounded context publishes the event. 

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

## Adding Support for Zero-cost Orders

There were three specific goals in making this change, all of which are
related:

1. Modify the **RegistrationProcess** workflow and related aggregates to
   handle orders with a zero cost.
2. Modify the navigation in the UI to skip the payment step when the
   total cost of the order is zero.
3. Ensure that the system functions correctly after the upgrade to V2
   with the old events as well as the new.

### Changes to the RegistrationProcess Workflow

Previously, the **RegistrationProcess** workflow sent a **ConfirmOrderPayment** command after it received notification from the UI that the registrant had completed the payment. Now, if there is a zero-cost order, the UI sends a **ConfirmOrder** command directly to the **Order** aggregate. If the order requires a payment, the **RegistrationProcess** workflow sends a **ConfirmOrder** command to the **Order** aggregate after it receives notification of a successful payment from the UI.

> **JanaPersona:** Notice that the name of the command has changed from
> **ConfirmOrderPayment** to **ConfirmOrder**. This reflects the fact
> that the order doesn't need to know anything about the payment, all it
> needs to know is that the order is confirmed. Similarly, there is a
> new **OrderConfirmed** event that is now used in place of the old
> **OrderPaymentConfirmed** event.

When the **Order** aggregate receives the **ConfirmOrder** command it raises an **OrderConfirmed** event. In addition to being peristed, this event is also handled by the following objects:

* The **OrderViewModelGenerator** class where it updates the sate of the
  order in the read-model.
* The **SeatAssignments** aggregate where it initializes a new
  **SeatAssignments** instance.
* The **RegistrationProcess** workflow where it triggers a command to
  commit the seat reservation.

### Changes to the UI

The main change in the UI is in the **RegistrationController** MVC 
controller class in the **SpecifyRegistrantAndPaymentDetails** action. 
Previously, this action method returned an 
**InitiateRegistrationWithThirdPartyProcessorPayment** action result; 
now, if the new **IsFreeOfCharge** property of the **Order** object is 
true, it returns a **CompleteRegistrationWithoutPayment** action result, 
otherwise it returns a 
**CompleteRegistrationWithThirdPartyProcessorPayment** action result. 

```Cs
[HttpPost]
public ActionResult SpecifyRegistrantAndPaymentDetails(AssignRegistrantDetails command, string paymentType, int orderVersion)
{
    ...

    var pricedOrder = this.orderDao.FindPricedOrder(orderId);
    if (pricedOrder.IsFreeOfCharge)
    {
        return CompleteRegistrationWithoutPayment(command, orderId);
    }

    switch (paymentType)
    {
        case ThirdPartyProcessorPayment:

            return CompleteRegistrationWithThirdPartyProcessorPayment(command, pricedOrder, orderVersion);

        case InvoicePayment:
            break;

        default:
            break;
    }

    ...
}
```

The **CompleteRegistrationWithThirdPartyProcessorPayment** redirects the 
user to the **ThirdPartyProcessorPayment** action and the 
**CompleteRegistrationWithoutPayment** method redirects the user 
directly to the **ThankYou** action. 

### Data Migration

The Conference Management bounded context stores order information from 
the Orders and Registrations bounded context in the **PricedOrders** 
table in its SQL database. Previously, the Conference Management bounded 
context received the **OrderPaymentConfirmed** event; now it receives 
the **OrderConfirmed** event that contains an additional 
**IsFreeOfCharge** property. This becomes a new column in the SQL 
database. 

> **Markus:** We didn't need to modify the existing data in this table
> during the migration because the default value for a boolean is
> **false**. All of the existing entries were created before the system
> supported zero-cost orders.

During the migration, any in-flight **ConfirmOrderPayment** commands 
could be lost because they are no longer handled by the **Order** 
aggregate. You should verify that none of these commands are currently 
on the command bus. 

> **PoePersona:** We need to plan carefully how to deploy the V2 release
> so that we can be sure that all the existing, in-flight
> **ConfirmOrderPayment** commands are processed by a worker role
> instance running the V1 release

The system persists the state of **RegistrationProcess** workflow 
instances to a SQL database table. There are no changes to the schema of 
this table. The only change you will see after the migration is an 
additional value in the **StateValue** column. This reflects the 
additional **PaymentConfirmationReceived** vlaue in the **ProcessState** 
enumeration in the **RegistrationProcess** class as shown in the 
following code sample: 

```Cs
public enum ProcessState
{
    NotStarted = 0,
    AwaitingReservationConfirmation = 1,
    ReservationConfirmationReceived = 2,
    PaymentConfirmationReceived = 3,
}
```

In the V1 release, the events that the event sourcing system persisted 
for the **Order** aggregate included the **OrderPaymentConfirmed** 
event. Therefore, the event store contains instances of this event type. 
In the V2 release, the **OrderPaymentConfirmed** is replaced with the 
**OrderConfirmed** event. 

The team decided for the V2 release not to introduce mapping and 
filtering events at the infrastructure level when events are 
deserialized. This means that the handlers must understand both the old 
and new events when the system replays these events from the event 
store. The following code sample shows this in the 
**SeatAssignmentsHandler** class: 

```Cs
static SeatAssignmentsHandler()
{
    Mapper.CreateMap<OrderPaymentConfirmed, OrderConfirmed>();
}

public SeatAssignmentsHandler(IEventSourcedRepository<Order> ordersRepo, IEventSourcedRepository<SeatAssignments> assignmentsRepo)
{
    this.ordersRepo = ordersRepo;
    this.assignmentsRepo = assignmentsRepo;
}

public void Handle(OrderPaymentConfirmed @event)
{
    this.Handle(Mapper.Map<OrderConfirmed>(@event));
}

public void Handle(OrderConfirmed @event)
{
    var order = this.ordersRepo.Get(@event.SourceId);
    var assignments = order.CreateSeatAssignments();
    assignmentsRepo.Save(assignments);
}
```

You can also see the same technique in use in the 
**OrderViewModelGenerator** class. 

The approach is slightly different in the **Order** class because this 
is one of the events that is persisted to the event store. The following 
code sample shows part of the **protected** constructor in the **Order** 
class: 

```Cs
protected Order(Guid id)
    : base(id)
{
    ...
    base.Handles<OrderPaymentConfirmed>(e => this.OnOrderConfirmed(Mapper.Map<OrderConfirmed>(e)));
    base.Handles<OrderConfirmed>(this.OnOrderConfirmed);
    ...
}
```

> **JanaPersona:** Handling the old events in this way was
> straightforward for this scenario because the only change was to the
> name of the event. It would be more complicated if the properties of
> the event changed as well. In the future, Contoso will consider doing
> the mapping in the infrastructure to avoid polluting the domain model
> with legacy events.

## Displaying Remaining Seats in the UI

There were three specific goals in making this change, all of which are
related:

1. Modify the system to include information about the
   number of remaining seats of each seat type in the conference
   read-model.
2. Modify the UI to display the number of remaining seats of each seat
   type.
3. Ensure that the system functions correctly after the upgrade to V2.

### Adding Information about Remaining Seat Quantities to the Read-Model

The information that the system needs to be able to display the number 
of remaining seats comes from two places. 

* The Conference Management bounded context raises the **SeatCreated**
  and **SeatUpdated** whenever the Business Customer creates new seat
  types or modifies seat quotas. 
* The **SeatsAvailability** aggregate in the Orders and Registrations
  bounded context raises the **SeatsReserved**,
  **SeatsReservationCancelled**, and **AvailableSeatsChanged** while a
  registrant is creating an order.
  
> **Note:** The **ConferenceViewModelGenerator** class does not use the
> **SeatCreated** and **SeatUpdated**

The **ConferenceViewModelGenerator** class in the Orders and 
Registrations bounded context now handles these events and uses them to 
calculate and store the information about seat type quantities in the 
read-model. The following code sample shows the relevant handlers in the 
**ConferenceViewModelGenerator** class: 

```Cs
public void Handle(AvailableSeatsChanged @event)
{
	this.UpdateAvailableQuantity(@event, @event.Seats);
}

public void Handle(SeatsReserved @event)
{
	this.UpdateAvailableQuantity(@event, @event.AvailableSeatsChanged);
}

public void Handle(SeatsReservationCancelled @event)
{
	this.UpdateAvailableQuantity(@event, @event.AvailableSeatsChanged);
}

private void UpdateAvailableQuantity(IVersionedEvent @event, IEnumerable<SeatQuantity> seats)
{
	using (var repository = this.contextFactory.Invoke())
	{
		var dto = repository.Set<Conference>().Include(x => x.Seats).FirstOrDefault(x => x.Id == @event.SourceId);
		if (dto != null)
		{
			if (@event.Version > dto.SeatsAvailabilityVersion)
			{
				foreach (var seat in seats)
				{
					var seatDto = dto.Seats.FirstOrDefault(x => x.Id == seat.SeatType);
					if (seatDto != null)
					{
						seatDto.AvailableQuantity += seat.Quantity;
					}
					else
					{
						Trace.TraceError("Failed to locate Seat Type read model being updated with id {0}.", seat.SeatType);
					}
				}

				dto.SeatsAvailabilityVersion = @event.Version;

				repository.Save(dto);
			}
			else
			{
				Trace.TraceWarning ...
			}
		}
		else
		{
			Trace.TraceError ...
		}
	}
}
```

The **UpdateAvailableQuantity** method compares the version on the event 
to current version of the read-model to detect possible duplicate 
messages. 

**MarkusPersona:** This check only detects duplicate messages, not out of sequence messages.

### Modifying the UI to Display Remaining Seat Quantities

Now, when the UI queries the conference read-model for a list of seat 
types, the list includes the currently available number of seats. The 
following code samples shows how the **RegistrationController** MVC 
controller uses the **AvailableQuantity** of the **SeatType** class: 

```Cs
private OrderViewModel CreateViewModel()
{
	var seatTypes = this.ConferenceDao.GetPublishedSeatTypes(this.ConferenceAlias.Id);
	var viewModel =
		new OrderViewModel
		{
			ConferenceId = this.ConferenceAlias.Id,
			ConferenceCode = this.ConferenceAlias.Code,
			ConferenceName = this.ConferenceAlias.Name,
			Items =
				seatTypes.Select(
					s =>
						new OrderItemViewModel
						{
							SeatType = s,
							OrderItem = new DraftOrderItem(s.Id, 0),
							AvailableQuantityForOrder = s.AvailableQuantity,
							MaxSelectionQuantity = Math.Min(s.AvailableQuantity, 20)
						}).ToList(),
		};

	return viewModel;
}
```

### Data Migration

The SQL table that holds the conference read-model data now has a new 
column to hold the version number that is used to check for duplicate 
events, and the SQL table that holds the seat type read-model data now 
has a new column to hold the available quantity of seats. 

As part of the data migration it is necessary to replay all of the 
events in the event store for each of the **SeatsAvailability** 
aggregates in order to correctly calculate the available quantities. 

## De-duplicating Command Messages

The system currently uses the Windows Azure Service Bus to transport messages. When the **SubscriptionReceiver** class creates a topic, it configures the topic to detect duplicate messages as shown in the following code sample:

```Cs
private void CreateTopicIfNotExists()
{
    var topicDescription =
        new TopicDescription(this.topic)
        {
            RequiresDuplicateDetection = true,
            DuplicateDetectionHistoryTimeWindow = TimeSpan.FromHours(1)
        };
    try
    {
        this.namespaceManager.CreateTopic(topicDescription);
    }
    catch (MessagingEntityAlreadyExistsException) { }
}
```

However, for the duplicate detection to work you must ensure that every message has a unique id. The following code sample shows the **MarkSeatsAsReserved** command:

```Cs
public class MarkSeatsAsReserved : ICommand
{
    public MarkSeatsAsReserved()
    {
        this.Id = Guid.NewGuid();
        this.Seats = new List<SeatQuantity>();
    }

    public Guid Id { get; set; }

    public Guid OrderId { get; set; }

    public List<SeatQuantity> Seats { get; set; }

    public DateTime Expiration { get; set; }
}
```

The **BuildMessage** method in the **CommandBus** class uses the command Id to create a unique message Id that the Windows Azure Service Bus can use to detect duplicates:

```Cs
private BrokeredMessage BuildMessage(Envelope<ICommand> command)
{
    var stream = new MemoryStream();
    ...

    var message = new BrokeredMessage(stream, true);
    if (!default(Guid).Equals(command.Body.Id))
    {
        message.MessageId = command.Body.Id.ToString();
    }

    ...

    return message;
}
```

# Testing 

Describe any special considerations that relate to testing for this bounded context.  
