## Chapter 9

# Technologies Used in the Reference Implementation (Chapter Title)

# Windows Azure Service Bus  

This section is not intended to provide an in-depth description of the 
Windows Azure Service Bus, rather it is intended to highlight those 
features that may prove useful in implementing the CQRS pattern and 
event sourcing. The section "Further Information" below, includes links 
to additional resources for you to learn more. 

The Windows Azure Service Bus provides a cloud-hosted, reliable 
messaging service. It operates in one of two modes: 

* **Relayed.** Relayed messaging provides a direct connection between
  clients who need to perform request/response messaging, one-way
  messaging, or peer-to-peer messaging.
* **Brokered.** Brokered messaging provides durable, asynchronous
  messaging between clients that are not necessarily connected at the
  same time. Brokered messaging supports both queue and
  publish/subscribe topologies.

In the context of CQRS and Event Sourcing, brokered messaging can 
provide the necessary messaging infrastructure for delivering commands 
and events reliably between elements of an application. The Windows 
Azure Service Bus also offers scalability in scenarios that must support 
high volumes of messages. 

## Queues

Windows Azure Service Bus queues provide a durable mechanism for senders 
to send one-way messages for delivery to a single consumer. 

Figure 1 shows how a queue delivers messages.

![Figure 1][fig1] 

**Windows Azure Service Bus Queue**

The following list describes some of the key characteristics of queues.

* Queues deliver messages on a First In, First Out (FIFO) basis.
* Multiple senders can send messages on the same queue.
* A queue can have multiple consumers, but an individual message is
  only consumed by one consumer. Multiple consumers compete for messages
  on the queue.
* Queues offer "temporal decoupling." Senders and consumer do not need
  to be connected at the same time.
 

## Topics and Subscriptions

Windows Azure Service Bus topics provide a durable mechanism for senders 
to send one-way messages for delivery to a multiple consumers. 

Figure 2 shows how a topic distributes messages.

![Figure 2][fig2] 

**Windows Azure Service Bus Topic**

The following list describes some of the key characteristics of topics.

* Topics deliver a copy of each message to each subscription.
* Multiple senders can publish messages to the same topic.
* Each subscription can have multiple consumers, but an individual
  message in a subscription is only consumed by one consumer. Multiple
  consumers compete for messages on the subscription.
* Topics offer "temporal decoupling." Senders and consumer do not need
  to be connected at the same time.
* Individual subscriptions support filters that limit the messages
  available through that subscription.

## Useful API Features

The following sections highlight some of the Windows Azure Service Bus 
API features that are used in the project. 

### Reading Messages

A consumer can use one of two modes to retrieve messages from queues or 
subscriptions: **ReceiveAndDelete** mode and **PeekLock** mode. 

In the **ReceiveAndDelete** mode, a consumer retrieves a message in a 
single operation: the Service Bus delivers the message to the consumer 
and marks the message as deleted. This is the simplest mode to use, but 
there is a risk that a message could be lost if the consumer fails 
between retrieving the message and processing it. 

In the **PeekLock** mode, a consumer retrieves a message in two steps: 
first, the consumer requests the message, the Service Bus delivers the 
message to the consumer and marks the message on the queue or 
subscription as locked; then, when the consumer has finished processing 
the message, it informs the Service Bus so that it can mark the message 
as deleted. In this scenario, if the consumer fails between retrieving 
the message and completing its processing, the message is re-delivered 
when the consumer restarts. A timeout ensures that locked messages 
become available again if the consumer does not complete the second 
step. 

In the **PeekLock** mode, it is possible that a message could be 
delivered twice in the event of a failure. This is known as *at least 
once* delivery. You must ensure that either the messages are idempotent, 
or add logic to the consumer to detect duplicate messages and ensure 
*exactly once* processing. Every message has a unique, unchanging Id 
which facilitates checking for duplicates. 

### Expiring Messages

When you create a **BrokeredMessage** object, you can specify an expiry 
time using the **ExpiresAtUtc** property or a time to live using the 
**TimeToLive** property. When a message expires you can specify either 
to send the message to a dead letter queue or discard it. 

### Delayed Message Processing

In some scenarios, you may want to send the message now, but to delay 
delivery until some future time. You can do this by using the 
**ScheduleEnqueueTimeUtc** property of the **BrokeredMessage** instance. 

## Serializing Messages

You must serialize your Command and Event objects if you are sending 
them over the Windows Azure Service Bus. 

The Contoso Conference Management System uses Json.NET serializer to 
serialize command and event messages. The team chose to use this 
serializer because of its flexibility and resilience to version changes. 

The following code sample shows the adapter class in the **Common** 
project that wraps the Json.NET serializer. 

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

# Unity Application Block

The MVC web application in the Contoso Conference Management System uses 
the Unity Application Block (Unity) dependency injection container. The 
**Global.asax.cs** file contains the type registrations for the command 
and event buses, and the repositories. This file also hooks up the MVC 
infrastructure to the Unity dependency resolver as shown in the 
following code sample: 


```Cs
protected void Application_Start()
{
    this.container = CreateContainer();
    RegisterHandlers(this.container);

    DependencyResolver.SetResolver(new UnityDependencyResolver(this.container));
	
	...
}
```

The following code sample shows the **UnityDependencyResolver** class:

```Cs
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Web.Mvc;
using Microsoft.Practices.Unity;

/// <summary>
/// This class allows ASP.NET MVC to use to Unity for resolving dependencies. 
/// For more information see http://msdn.com/library/system.web.mvc.idependencyresolver
/// </summary>
public class UnityDependencyResolver : IDependencyResolver
{
    private readonly IUnityContainer _unity;

    public UnityDependencyResolver(IUnityContainer unity)
    {
        _unity = unity;
    }

    public object GetService(Type serviceType)
    {
        try
        {
            return _unity.Resolve(serviceType);
        }
        catch (ResolutionFailedException)
        {
            // By definition of IDependencyResolver contract,
            // this should return null if it cannot be found.
            // http://msdn.com/library/system.web.mvc.idependencyresolver.getservice
            Debug.WriteLine(string.Format("Unable to resolve request for {0}, returning null", serviceType.Name));
            return null;
        }
    }

    public IEnumerable<object> GetServices(Type serviceType)
    {
        try
        {
            return _unity.ResolveAll(serviceType);
        }
        catch (ResolutionFailedException)
        {
            // By definition of IDependencyResolver contract,
            // this should return an empty collection if it cannot be found.
            // http://msdn.com/library/system.web.mvc.idependencyresolver.getservices
            Debug.WriteLine(string.Format("Unable to resolve request for a collection of {0}, returning an empty collection", serviceType.Name));
            return new object[0];
        }
    }
}
```

The MVC controller classes no longer have parameter-less constructors. 
The following code sample shows the constructor from the 
**RegistrationController** class: 

```Cs
private ICommandBus commandBus;
private Func<IViewRepository> repositoryFactory;

public RegistrationController(ICommandBus commandBus, [Dependency("registration")]Func<IViewRepository> repositoryFactory)
{
    this.commandBus = commandBus;
    this.repositoryFactory = repositoryFactory;
}
```


[fig1]:           images/Reference_09_ServiceBusQueue.png?raw=true
[fig2]:           images/Reference_09_ServiceBusTopic.png?raw=true

[sb]:             http://msdn.microsoft.com/en-us/library/ee732537.aspx
[sbperf]:         http://msdn.microsoft.com/en-us/library/hh528527.aspx
[sbpatterns]:     http://msdn.microsoft.com/en-us/library/hh410103.aspx
[jsonnet]:		  http://james.newtonking.com/pages/json-net.aspx