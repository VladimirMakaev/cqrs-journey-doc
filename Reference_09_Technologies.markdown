## Chapter 9

# Technologies Used in the Reference Implementation (Chapter Title)

# Windows Azure Service Bus  

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Cover Brokered Messages, Queues, Topics, Subscriptions
  </span>
</div> 
This section is not intended to provide an in-depth description of the Windows Azure Service Bus, rather it is intended to highlight those features that may prove useful in implementing the CQRS pattern and event sourcing. The section "Further Information" below, includes links to additional resources for you to learn more.

The Windows Azure Service Bus provides a cloud-hosted, reliable messaging service. It operates in one of two modes:

* **Relayed.** Relayed messaging provides a direct connection between clients who need to perform request/response messaging, one-way messaging, or peer-to-peer messaging.
* **Brokered.** Brokered messaging provides durable, asynchronous messaging between clients that are not necessarily connected at the same time. Brokered messaging supports both queue and publish/subscribe topologies.

In the context of CQRS and Event Sourcing, brokered messaging can provide the necessary messaging infrastructure for delivering commands and events reliably between elements of an application. The Windows Azure Service Bus also offers scalability in scenarios that must support high volumes of messages.

## Queues

Windows Azure Service Bus queues provide a durable mechanism for senders to send one-way messages for delivery to a single consumer. 

Figure 1 shows how a queue delivers messages.

![Figure 1][fig1] 

**Windows Azure Service Bus Queue**

The following list describes some of the key characteristics of queues.

* Queues deliver messages on a First In, First Out (FIFO) basis.
* Multiple senders can send messages on the same queue.
* A queue can have multiple consumers, but an individual message is only consumed by one consumer. Multiple consumers compete for messages on the queue.
* Queues offer "temporal decoupling." Senders and consumer do not need to be connected at the same time. 

## Topics and Subscriptions

Windows Azure Service Bus topics provide a durable mechanism for senders to send one-way messages for delivery to a multiple consumers. 

Figure 2 shows how a topic distributes messages.

![Figure 2][fig2] 

**Windows Azure Service Bus Topic**

The following list describes some of the key characteristics of topics.

* Topics deliver a copy of each message to each subscription.
* Multiple senders can publish messages to the same topic.
* Each subscription can have multiple consumers, but an individual message in a subscription is only consumed by one consumer. Multiple consumers compete for messages on the subscription.
* Topics offer "temporal decoupling." Senders and consumer do not need to be connected at the same time.
* Individual subscriptions support filters that limit the messages available through that subscription.

## Useful API Features

The following sections highlight some of the Windows Azure Service Bus API features that are used in the project.

### Reading Messages

A consumer can use one of two modes to retrieve messages from queues or subscriptions: **ReceiveAndDelete** mode and **PeekLock** mode.

In the **ReceiveAndDelete** mode, a consumer retrieves a message in a single operation: the Service Bus delivers the message to the consumer and marks the message as deleted. This is the simplest mode to use, but there is a risk that a message could be lost if the consumer fails between retrieving the message and processing it.

In the **PeekLock** mode, a consumer retrieves a message in two steps: first, the consumer requests the message, the Service Bus delivers the message to the consumer and marks the message on the queue or subscription as locked; then, when the consumer has finished processing the message, it informs the Service Bus so that it can mark the message as deleted. In this scenario, if the consumer fails between retrieving the message and completing its processing, the message is re-delivered when the consumer restarts. A timeout ensures that locked messages become available again if the consumer does not complete the second step.

In the **PeekLock** mode, it is possible that a message could be delivered twice in the event of a failure. This is known as *at least once* delivery. You must ensure that either the messages are idempotent, or add logic to the consumer to detect duplicate messages and ensure *exactly once* processing. Every message has a unique, unchanging Id which facilitates checking for duplicates.

### Expiring Messages

When you create a **BrokeredMessage** object, you can specify an expiry time using the **ExpiresAtUtc** property or a time to live using the **TimeToLive** property. When a message expires you can specify either to send the message to a dead letter queue or discard it.

### Delayed Message Processing

In some scenarios, you may want to send the message now, but to delay delivery until some future time. You can do this by using the **ScheduleEnqueueTimeUtc** property of the **BrokeredMessage** instance.

## Serializing Messages

You must serialize your Command and Event objects if you are sending them over the Windows Azure Service Bus.

The Contoso Conference Management System uses Json.NET serializer to serialize messages. The following code sample shows the adapter class in the **Common** project that wraps the Json.NET serializer.

```Cs
public class JsonSerializerAdapter : ISerializer
{
    private JsonSerializer serializer;

    public JsonSerializerAdapter(JsonSerializer serializer)
    {
        this.serializer = serializer;
    }

    public void Serialize(Stream stream, object graph)
    {
        var writer = new JsonTextWriter(new StreamWriter(stream));

        this.serializer.Serialize(writer, graph);

        // We don't close the stream as it's owned by the message.
        writer.Flush();
    }

    public object Deserialize(Stream stream)
    {
        var reader = new JsonTextReader(new StreamReader(stream));

        return this.serializer.Deserialize(reader);
    }
}
```


## Further Information

For general information about the Windows Azure Service Bus, see 
[Service Bus][sb] on MSDN. 

For more information about Service Bus topologies and patterns, see
[Overview of Service Bus Messaging Patterns][sbpatterns] on MSDN.

For information about scaling the Windows Azure Service Bus 
infrastructure, see [Best Practices for Performance Improvements Using 
Service Bus Brokered Messaging][sbperf] on MSDN. 

For information about Json.NET, see [Json.NET][jsonnet].

# JSON Serializer

You must serialize your Command and Event objects if you are sending them over the Windows Azure Service Bus.


[fig1]:           images/Reference_09_ServiceBusQueue.png?raw=true
[fig2]:           images/Reference_09_ServiceBusTopic.png?raw=true

[sb]:             http://msdn.microsoft.com/en-us/library/ee732537.aspx
[sbperf]:         http://msdn.microsoft.com/en-us/library/hh528527.aspx
[sbpatterns]:     http://msdn.microsoft.com/en-us/library/hh410103.aspx
[jsonnet]:		  http://james.newtonking.com/pages/json-net.aspx