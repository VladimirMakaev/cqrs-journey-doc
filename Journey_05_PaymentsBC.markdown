## Chapter 5
# Preparing for the V1 Release  

*Adding functionality and refactoring in preparation for the V1 release.*

# A Description of the Contoso Conference Management V1 Release

This chapter describes the changes made by the team to prepare for the 
first production release of the Contoso Conference Management System. 
This work includes some refactoring and additions to the Orders and 
Registrations bounded context that was introduced in the previous two 
chapters as well as a new Conference Management bounded context. 

One of the key refactorings undertaken by the team during this phase of 
the journey was to introduce event sourcing into the Orders and 
Registrations bounded context. 

One of the anticipated benefits from implementing the CQRS pattern is 
that it will help to manage change in a complex system. Having a V1 
release during the CQRS journey will help the team to evaluate how the 
CQRS pattern and event sourcing deliver these benefits when they move 
forward from the V1 release to the next production release of the 
system. The following chapters will describe what happens after the V1 
release. 

This chapter also describes the Metro-style UI that the team added to 
the public web-site during this phase and includes a discussion of 
task-based UIs. 

## Working Definitions for this Chapter 

Outline any working definitions that were adopted for this chapter. 

### Access Code

When a Business Customer creates a new Conference, the system generates 
a five character Access Code and sends it by email to the Business 
Customer. The Business Customer can use his email address and the Access 
Code on the Conference Management Web Site to retrieve the conference 
details from the system at a later date. The system uses access codes 
instead of passwords to avoid the overhead for the Business Customer of 
setting up an account with the system. 

## User Stories 

The team implemented the user stories listed below during this phase of 
the project. 

### Conference Management User Stories

The **Business Customer** represents the organization that is using the 
conference management system to run its conference. 

A **Seat** represents a space at a conference or access to a specific 
session at the conference such as a cocktail party, a tutorial, or a 
workshop. 

A business customer can create new conferences and manage them. After a 
business customer creates a new conference, he can access the details of 
the conference by using his email address and conference locator access 
code. The system generates the access code when the business customer 
creates the conference. 

The business customer can specify the following information about a 
conference: 

* The name, description, and slug (part of the URL used to access the
  conference).
* The start and end dates of the conference.
* The different types and quotas of seats available at the conference.

Additionally, the business customer can control the visibility of the 
conference on the public web-site by either publishing or un-publishing 
the conference. 

The business customer can use the conference management web-site to view 
a list of attendees. 

### Ordering and Registration User Stories

After a registrant has purchased seats at a conference, she can assign 
attendees to those seats. The system stores the name and contact details 
for each attendee. 

## Architecture 

What are the key architectural features? Server-side, UI, multi-tier, cloud, etc.

## Conference Management Bounded Context

The Conference Management bounded context is a simple two-tier, CRUD-style web application. It is implemented using MVC 4 and Entity Framework.

**JanaPersona:** The team implemented this bounded context after it implemented the public Conference Management web-site that uses MVC 3.

# Patterns and Concepts 

* What are the primary patterns or approaches that have been adopted for this bounded context? (CQRS, CQRS/ES, CRUD, ...) 

* What were the motivations for adopting these patterns or approaches for this bounded context? 

* What trade-offs did we evaluate? 

* What alternatives did we consider? 

## Event Sourcing

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Add brief summary of event sourcing with links to Reference Guide.
  </span>
</div> 

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

## CRUD

You should not use the CQRS pattern as part of your top-level 
architecture: you should implement the pattern only in those bounded 
contexts where it brings clear benefits. In the Contoso Conference 
Management System, the conference management bounded context is a 
relatively simple, stable, and low volume element of the overall system. 
Therefore the team decided that they would implement this bounded 
context using a traditional two-tier, CRUD-style architecture. 

## Integration between Bounded Contexts

The Conference Management bounded context needs to integrate with the 
Orders and Registrations bounded context. For example, if the business 
customer changes the quota for a seat type in the Conference Management 
bounded context, this change needs to be propagated to the Orders and 
Registrations bounded context. Also, if a registrant adds a new attendee 
to a conference, the business customer must be able to view details of 
the attendee in the list in the Conference Management web-site. 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Add details of integration options: db-centric, events, read-model.
  </span

# Implementation Details 

Describe significant features of the implementation with references to the code. Highlight any specific technologies (including any relevant trade-offs and alternatives). 

Provide significantly more detail for those BCs that use CQRS/ES. Significantly less detail for more "traditional" implementations such as CRUD. 

## The Conference Management Bounded Context

The Conference Management Bounded Context that enables a Business Customer to define and  manage conferences is implemented using a simple two-tier, CRUD-style application using MVC 4.

In the Visual Studio solution, the **Conference** project contains the model code, and the **Conference.Web** project contains the MVC views and controllers.

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Add details of intregration with other bounded contexts.
  </span>
</div> 

# Running the Applications

You can run the Contoso Conference Management System in two modes: 
either deployed to Windows Azure, or running locally. 

If you deploy the V1 Release of the Contoso Conference Management System 
to Windows Azure, then the application uses Windows Azure Service Bus to 
provide its messaging infrastructure using brokered messages. If you run 
the application locally, then it runs using the Windows Azure Compute 
Emulator and uses the **MemoryEventBus** and **MemoryCommandBus** 
classes in the Common project to provide messaging services for events.

## Running the Contoso Conference Management System Locally

The following procedure describes how to configure the Contoso 
Conference Management System to run locally without using Windows Azure. 
Running the application in this way is useful if you want to quickly 
explore how the application works without configuring Windows Azure 
accounts. 

**To Run the Contoso Conference Management System Locally**

1. In Visual Studio, open the **Conference** solution.
2. On the **Build** menu, click **Configuration Manager**.
3. In the **Active solution configuration** drop-down list, select
   **DebugLocal**, then click **Close**.
4. You can now build and run the solution locally.

## Running the Contoso Conference Management System on Windows Azure

The following procedure describes how to configure the Contoso 
Conference Management System to run on Windows Azure. 
Running the application in this way is more realistic, but you must
complete some additional configuration steps. 

**To Run the Contoso Conference Management System on Windows Azure**

1. In Visual Studio, open the **Conference** solution.
2. On the **Build** menu, click **Configuration Manager**.
3. In the **Active solution configuration** drop-down list, select
   **Debug**, then click **Close**.
4. In **Solution Explorer**, in the **Azure** folder, right-click the
   **Settings.Template.xml" file, and then click **Rename**.
5. Rename the the **Settings.Template.xml** file to **Settings.xml**.
6. In **Solution Explorer**, double-click the **Settings.xml** file to
   open it in the editor.
7. Follow the instructions in the file to enter your Windows Azure
   account details.
8. On the **File** menu, click **Save Settings.xml**.
4. You can now build the solution and and deploy it to Windows Azure.

> **Note:** Other projects in the solution have links to the
> **Settings.xml** file in order to access the Windows Azure account
>  information.

## Conditional Compilation Notes
 
The following code sample from the **Global.asax.cs** file in the web 
projects shows how the project uses conditional compilation to select 
between the implementations.

```Cs
#if LOCAL
            container.RegisterType<ICommandBus, MemoryCommandBus>(new ContainerControlledLifetimeManager());
            container.RegisterType<ICommandHandlerRegistry, MemoryCommandBus>(new ContainerControlledLifetimeManager(), new InjectionFactory(c => new MemoryCommandBus()));
            container.RegisterType<IEventBus, MemoryEventBus>(new ContainerControlledLifetimeManager());
            container.RegisterType<IEventHandlerRegistry, MemoryEventBus>(new ContainerControlledLifetimeManager(), new InjectionFactory(c => new MemoryEventBus()));
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

            container.RegisterInstance<ICommandBus>(commandBus);
            container.RegisterInstance<IEventBus>(eventBus);
            container.RegisterInstance<ICommandHandlerRegistry>(commandProcessor);
            container.RegisterInstance(commandProcessor);
            container.RegisterInstance<IEventHandlerRegistry>(eventProcessor);
            container.RegisterInstance(eventProcessor);
#endif
```

> **MarkusPersona:** The code sample also shows how the application uses
> the [Unity Application Block][unity] dependency injection container.  
> The solution only uses Unity in the web application, not in the domain
> classes.  
> For more information, see [Technologies Used in the Reference
> Implementation][r_chapter9] in the Reference Guide.

# Testing 

Describe any special considerations that relate to testing for this bounded context.  

[r_chapter9]:     		Reference_09_Technologies.markdown

[inductiveui]:			http://msdn.microsoft.com/en-us/library/ms997506.aspx
[metroux]:              http://msdn.microsoft.com/en-us/library/windows/apps/hh465424.aspx
[unity]:				http://msdn.microsoft.com/en-us/library/ff647202.aspx