## Chapter 1
# The Contoso Conference Management System 

*The starting point: Where have we come from, what are we taking, and who is coming with us?*

This chapter introduces a fictitious company named Contoso. It describes 
Contoso's plans to launch the Contoso Conference Management System, a 
new, online service that will enable other companies or individuals to 
organize and manage their own conferences and events. The chapter 
describes, at a high-level, some of the functional and non-functional 
requirements of the new system, and why Contoso wants to implement parts 
of the new system using the CQRS pattern and event sourcing. As with any 
company considering this process, there are many issues to consider and 
challenges to be met, particularly because this is the first time 
Contoso has used both the CQRS pattern and event sourcing. The chapters 
that follow this one show, step-by-step, how Contoso designed and 
built its conference management application. 

The chapter also introduces a panel of fictional experts to comment 
on the development efforts. 

# The Contoso Corporation 

Contoso is a startup ISV company of approximately 20 employees that 
specializes in developing solutions using Microsoft&reg; technologies. 
The developers at Contoso are knowledgeable about various Microsoft 
products and technologies, including the .NET Framework, ASP.NET MVC, 
and Windows Azure. Some of the developers have previous experience of 
using the Domain-Driven Design (DDD) approach, but none of them have 
used the CQRS pattern previously. 

The Conference Management System application is one of the first 
innovative online services that Contoso wants to take to market. As a 
startup, Contoso wants to develop and launch these services with a 
minimal investment in hardware and IT personnel. Contoso wants to be 
quick to market in order to start growing market share, and cannot 
afford the time to implement all of the planned functionality in the 
first releases. Therefore, it is important that the architecture it 
adopts can easily accommodate changes and enhancements with minimal 
impact on existing users of the system. Contoso has chosen to deploy the 
application on Windows Azure in order to take advantage of its ability 
to scale applications as demand grows. 

# Who Is Coming With Us On the Journey? 

As mentioned earlier, this guide and the accompanying RI describe a CQRS 
journey. A panel of experts comments on our development efforts as we go. 
This panel includes a CQRS expert, a software architect, a developer, a 
domain expert, an IT Pro, and a business manager. They will all comment 
from their own perspectives. 

![Bharath][personabharath]
Bharath is a CQRS expert. He checks that a CQRS based solution will work 
for a company and will provide tangible benefits. He is a cautious 
person, for good reason. 

> "Defining the CQRS pattern is easy. Realizing the benefits that
> implementing the CQRS pattern can offer is not always so
> straightforward."

![Jana][personajana]
Jana is a software architect. She plans the overall structure of an 
application. Her perspective is both practical and strategic. In other 
words, she considers not only what technical approaches are needed 
today, but also what direction a company needs to consider for the 
future. Jana has worked on projects that used the Domain-Driven Design 
approach.

> "It's not easy to balance the needs of the company, the users, the IT
> organization, the developers, and the technical platforms we rely on."
  
![Markus][personamarkus]
Markus is a software developer who is new to the CQRS pattern. He is 
analytical, detail-oriented, and methodical. He's focused on the task at 
hand, which is building a great application. He knows that he's the 
person who's ultimately responsible for the code. 

> "I don't care what architecture you want to use for the application,
> I'll make it work."

![Carlos][personacarlos]
Carlos is the domain expert. He understands all the ins and outs of 
conference management. He has worked in a number of organizations that 
help people to run conferences. He has also worked in a number of 
different roles: sales and marketing, conference management, and 
consultant. 

> "I want to make sure that the team understands how this business
> works so that we can deliver a world-class online conference 
> management system."

![Poe][personapoe]
Poe is an IT professional who's an expert in deploying and running 
applications in the cloud. Poe has a keen interest in practical 
solutions; after all, he's the one who gets paged at 3:00 AM when 
there's a problem. 

> "Running complex applications in the cloud involves different
> challenges from managing on-premises applications. I want to make
> sure our new Conference Management System meets our published SLAs."

![Beth][personabeth]
Beth is a business manager. She helps companies to plan how their 
business will develop. She understands the market that the company 
operates in, the resources that the company has available, and the goals 
of the company. She has both a strategic view, and an interest in the 
day-to-day operations of the company. 

> "Organizations face many conflicting demands on their resources. I 
> want to make sure that our company balances those demands and adopts a 
> business plan that will make us successful in the medium and long 
> term."

If you have a particular area of interest, look for notes provided by 
the specialists whose interests align with yours. 

# The Conference Management System 

This section describes the Contoso Conference Management System as the  
team envisaged it at the start of the journey. The team has not 
used the CQRS pattern before; therefore the system that is delivered at 
the end of our journey may not match this description exactly because: 

* What we learn as we go may impact on what we plan to deliver.
* It is more difficult to estimate what can be achieved in the available
  time.

## Overview of the System

Contoso plans to build an online conference management system that will 
enable Contoso's customers to plan and manage conferences that are held 
at a physical location. The system will enable Contoso's customers to: 

* Manage the sale of different seat types for the conference.
* Manage the conference on site, including badge printing and attendee
  lists.
* Create a conference and define characteristics of that conference.
* Plan the tracks, sessions, and speakers that make up a conference.

The Contoso Conference Management System will be a multi-tenant, 
cloud-hosted application. Business Customers will need to register with 
the system before they can create and manage their conferences. 

### Selling Seats for a Conference

The Business Customer defines how many seats are available for the 
conference. The business customer may also specify events at a 
conference such as workshops, receptions, and premium sessions for which 
attendees must have a separate ticket. The business customer will also 
define how many seats are available for these events. 

The system manages the sale of seats to ensure that the conference and 
sub-events are not oversubscribed. This part of the system will also 
operate wait-lists so that if other attendees cancel, then their seats 
can be re-allocated. 

The system will require that the names of the attendees are associated 
with the purchased seats so that an on-site system can print badges for
the attendees when they arrive at the conference. 

### Creating a Conference

A Business Customer can create new conferences and manage information 
about the conference such as its name, description, and dates. The 
Business Customer can also make a conference visible on the Contoso 
Conference Management System web site by publishing it, or hide it by 
un-publishing it. 

Additionally the Business Customer defines the seat types and available 
quantity of each seat type for the conference. 

Contoso also plans to enable the Business Customer to specify the 
following characteristics of a conference: 

* Will the paper submission process require reviewers?
* What will be the fee structure for paying Contoso?
* Assigning key personnel such as the Program Chair and the Event
  Planner.

### Building a Program

The business customer defines the conference program. This consists of 
the following tasks: 

* Identifying Tracks and Track Chairs.
* Identifying Reviewers.
* Making calls for submissions.
* Reviewing the papers and assigning speakers to sessions.
* Finalizing Track programs.

## Non-functional Requirements

Contoso has two major non-functional requirements for its Conference 
Management System and it hopes that the CQRS pattern will help it to 
meet them. 

### Scalability

The Conference Management System will be hosted in the cloud, and one of 
the reasons Contoso chose a cloud platform was its scalability and 
potential for elastic scalability. 

Although cloud platforms such as Windows Azure enable you to scale 
applications by adding (or removing) role instances, you must still 
design your application to be scalable. By splitting responsibility 
for the application's read operations and write operations into separate 
objects, the CQRS pattern gives you the ability to split your 
application's read and write operations into separate Windows Azure 
roles that you can scale independently of each other. This recognizes 
the fact that for many applications the number of read operations vastly 
exceeds the number of write operations. This gives Contoso the 
opportunity to scale the Conference Management System more efficiently, 
and make better use of the Windows Azure role instances that it uses. 

### Flexibility

The market that the Contoso Conference Management System operates in is 
very competitive, and very fast moving. In order to compete, Contoso 
must be able to quickly and cost effectively adapt the Conference 
Management System to changes in the market. This requirement for 
flexibility breaks down into a number of related aspects: 

* Contoso must be able to evolve the system to meet new requirements 
  and to respond to changes in the market. 

* The system must be able to run multiple versions of its software 
  simultaneously in order to support customers who are in the middle of 
  a conference and who do not wish to upgrade to a new version 
  immediately. Other customers may wish to migrate their existing 
  conference data to a new version of the software as it becomes 
  available.
  
> **PoePersona:** This is a big challenge: keeping the system running
> for all our customers while we perform upgrades with no downtime.

* Contoso intends the software to last for at least five years. It 
  must be able to accomodate significant changes over that period. 

* Contoso does not want the complexity of some parts of the system to 
  become a barrier to changes. 

* Contoso would like to be able to use different developers for 
  different elements of the system, using cheaper developers for simpler 
  tasks and restricting its use of more expensive and experienced 
  developers for the more critical aspects of the system. 

> **BharathPersona:** There is some debate in the CQRS community 
> about whether, in practice, you can use different development teams 
> for different parts of the CQRS pattern implementation. 

# Beginning the Journey

The next chapter is the start of our CQRS journey. It provides more 
information about the Contoso Conference Management System and describes 
some of the high-level parts of the system. Subsequent chapters describe 
the stages of journey as Contoso implements the Conference Management 
System. 

[personabharath]: images/PersonaBharath.png?raw=true
[personajana]:    images/PersonaJana.png?raw=true
[personamarkus]:  images/PersonaMarkus.png?raw=true
[personacarlos]:  images/PersonaCarlos.png?raw=true
[personapoe]:     images/PersonaPoe.png?raw=true
[personabeth]:    images/PersonaBeth.png?raw=true