## Chapter 5
# Designing and Implementing the Payments Bounded Context  

*Our first stopping point.*

# A Description of the Payments Bounded Context 

Summary description of this Bounded Context. What is its relative 
importance/significance in the domain – is it core, how does it relate 
to the other bounded contexts? 

Will also include a discussion of task-based UIs.

## Working Definitions for this Chapter 

Outline any working definitions that were adopted for this chapter. 

## User Stories 

What were the key user stories addressed in this chapter?  

## Architecture 

What are the key architectural features? Server-side, UI, multi-tier, cloud, etc. 

# Patterns and Concepts 

* What are the primary patterns or approaches that have been adopted for this bounded context? (CQRS, CQRS/ES, CRUD, ...) 

* What were the motivations for adopting these patterns or approaches for this bounded context? 

* What trade-offs did we evaluate? 

* What alternatives did we consider? 

## Task-based UI

The design of UIs has improved greatly over the last decade: 
applications are easier to use, more intuitive, and simpler to navigate 
than they were before. Examples of guidelines for UI designers are the 
[Microsoft Inductive User Interface Guidelines][inductiveui] and the [UX 
guidelines for Metro style apps][metroux]. 

Another factor that affects the design and usability of the UI is how 
the UI communicates with the rest of the application. If the application 
is based on a CRUD-style architecture, this can leak through to the UI. 
If the developers focus on CRUD-style operations, this can result in a 
UI as shown in the first screen design in Figure 1. 

![Figure 1][fig1]

**Example UIs for conference registration**

On the first screen, the labels on the buttons reflect the underlying 
CRUD operations that the system will perform when the user clicks the 
**Submit** button. The first screen also requires the user to apply some 
deductive knowledge about how the screen and the application function. 
For example, the function of the **Add** button is not immediately 
apparent. 

A typical implementation behind the first screen will use a data 
transfer object (DTO) to exchange data between the back-end and the UI. 
The UI will request data from the back-end that will arrive encapsulated 
in a DTO, it will modify the data in the DTO, and then return the DTO 
to the back-end. The back-end will use the DTO to figure out what CRUD 
operations it must perform on the underlying datastore. 

The second screen is more explicit about what is happening in terms of 
the business process: the user is selecting quantities of seat types as 
a part of the conference registration task. Thinking about the UI in 
terms of the task that the user is performing makes it easier to relate 
the UI to the write-model in your implementation of the CQRS pattern. 
The UI can send commands to the write-side, and those commands are a 
part of the domain model on the write-side. In a bounded context that 
implements the CQRS pattern, the UI typically queries the read-side and 
receives a DTO, and sends commands to the write-side. 

![Figure 2][fig2]

**Task-based UI flow**

Figure 2 shows a sequence of pages that enable the registrant to 
complete the "purchase seats at a conference" task. On the first page, 
the registrant selects the type and quantity of seats. On the second 
page, the registrant can review the seats she has reserved, enter her 
contact details, and complete the necessary payment details. The system 
then redirects the registrant to a payment provider, and if the payment 
completes successfully, the system displays the third page. The third 
page shows a summary of the order and provides a link to pages where the 
registrant can start additional tasks. 

The sequence shown in Figure 2 is deliberately simplified in order to 
highlight the roles of the commands and queries in a task-based UI. For 
example, the real flow includes pages that the system will display based 
on the payment type selected by the registrant, and error pages that the 
system displays if the payment fails. 

> **BharathPersona:** You don't always need to use task-based UIs. In
> some scenarios, simple CRUD-style UIs work well. You must evaluate
> whether benefits of task-based UIs outweigh the additional 
> implementation effort of a task-based UI. Very often, the bounded
> contexts where you choose to implement the CQRS pattern, are also the
> bounded contexts that benefit from task-based UIs because of the 
> more complex business logic and more complex user interactions. 


# Implementation Details 

Describe significant features of the implementation with references to the code. Highlight any specific technologies (including any relevant trade-offs and alternatives). 

Provide significantly more detail for those BCs that use CQRS/ES. Significantly less detail for more "traditional" implementations such as CRUD. 

# Running the Applications

You can run the Contoso Conference Management System in two modes: either deployed to Windows Azure, or running locally.

If you deploy the V1 Release of the Contoso Conference Management System to Windows Azure, then the application uses Windows Azure Service Bus to provide its messaging infrastructure using brokered messages. If you run the application locally, then it runs using the Windows Azure Compute Emulator and uses the **MemoryEventBus** and **MemoryCommandBus** classes in the Common project to provide messaging services for events. The following code sample from the **Global.asax.cs** file in the web projects shows how the project uses conditional compilation to select between the implementations.

```Cs
#if LOCAL
    var commandBus = new MemoryCommandBus();
    var commandProcessor = commandBus;
    var eventBus = new MemoryEventBus();
    var eventProcessor = eventBus;
#else
    var serializer = new JsonSerializerAdapter(JsonSerializer.Create(new JsonSerializerSettings
    {
        // Allows deserializing to the actual runtime type
        TypeNameHandling = TypeNameHandling.Objects,
        // In a version resilient way
        TypeNameAssemblyFormat = System.Runtime.Serialization.Formatters.FormatterAssemblyStyle.Simple
    }));

    var settings = MessagingSettings.Read(HttpContext.Current.Server.MapPath("bin\\Settings.xml"));
    var commandBus = new CommandBus(new TopicSender(settings, "conference/commands"), new MetadataProvider(), serializer);
    var eventBus = new EventBus(new TopicSender(settings, "conference/events"), new MetadataProvider(), serializer);

    var commandProcessor = new CommandProcessor(new SubscriptionReceiver(settings, "conference/commands", "all"), serializer);
    var eventProcessor = new EventProcessor(new SubscriptionReceiver(settings, "conference/events", "all"), serializer);
#endif
```



# Testing 

Describe any special considerations that relate to testing for this bounded context.  

[inductiveui]:			http://msdn.microsoft.com/en-us/library/ms997506.aspx
[metroux]:              http://msdn.microsoft.com/en-us/library/windows/apps/hh465424.aspx