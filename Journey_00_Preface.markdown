## Chapter 0
# Preface

_Why are we embarking on this journey?_

# What Are the Motivations for Creating This Guidance Now?

# How is This Guidance Structured?

There are two, closely related, parts to this guidance: 

* A working, sample, reference implementation (RI) that is intended to 
  illustrate many of the concepts related to the Command Query 
  Responsibility Segregation (CQRS) and Event Sourcing (ES) approaches 
  to developing complex enterprise applications. 

* This written guidance that is intended to complement the RI by 
  describing how it works, what decisions were made during its 
  development, and what trade-offs were considered. 

This written guidance is itself split into two distinct sections that 
you can read independently: description of the journey we took as we 
learned about CQRS, and a collection of CQRS reference materials. The 
map in figure 1 illustrates the relationship between the two sections: a 
journey with some defined stopping points that enables us to explore a 
space. 

![Figure 1][fig1]

**A CQRS journey**

## A CQRS Journey

This section is closely related to the RI and the chapters follow the 
chronology of the project to develop the RI. Each chapter describes 
relevant features of the domain model, infrastructure elements, 
architecture, and UI that the team were concerned with during that phase 
of the project. Some parts of the system are discussed in several 
chapters and this reflects the fact that the team revisited certain 
areas during later stages. Each of these chapters will discuss how and 
why particular CQRS patterns and concepts apply to the design and 
development of particular bounded contexts, will describe the 
implementation, and will highlight any implications for testing. 

There are also chapters that look at the big picture. For example, there 
is a chapter that explains the rationale for splitting the RI into the 
bounded contexts we chose, another chapter analyzes the implications of 
our approach for versioning the system, and other chapters looks at how 
the different bounded contexts in the RI communicate with each other. 

This section describes our journey as we learned about CQRS, and how we 
applied that learning to the design and implementation of the RI. It is 
not prescriptive guidance and is not intended to illustrate the only way 
to apply the CQRS approach to our RI. We have tried wherever possible to 
capture alternative viewpoints through consultation with our advisors 
and to explain why we took particular decisions. You may disagree with 
some of those decisions; please let us know. 

This section of the written guidance makes frequent cross-references to 
the material in the second section for readers who wish to explore any 
of the concepts or patterns in more detail. 

## CQRS Reference

The second section of the written guidance is a collection of reference 
material collated from many sources. It is not the definitive 
collection, but should contain enough material to help you to understand 
the core patterns, concepts, and language of CQRS. 

The following is a list of the chapters that comprise both sections of 
the written guidance: 


<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	To do
  </span>
</div> 

## What Were the Criteria for Selecting the Domain for the Reference Implementation?

Before embarking on our journey, we needed to have an outline of the 
route we planned to take and an idea of what the final destination 
should be. We needed to select an appropriate domain for the RI. 

We engaged with the community and our advisory board to help us choose a 
domain that would showcase as many of the features and concepts of CQRS 
as possible. To help us to select between our candidate domains we used 
criteria in the following list. The domain selected should be: 

* **Non-trivial.** The domain must be complex enough to exhibit real 
problems, but at the same time common enough for most people to 
understand without weeks of study. The problems should involve dealing 
with temporal data, stale data, receiving out of order events, and 
versioning. The domain should enable us to illustrate solutions using 
event sourcing, sagas, and event merging. 

* **Collaborative.** The domain must contain collaborative elements where 
multiple actors can operate simultaneously on shared data. 

* **End-to-end.** We wanted to be able illustrate the concepts and 
patterns in action from the back-end data store through to the user 
interface (UI). This might include disconnected mobile and smart 
clients. 

* **Cloud friendly.** We wanted to have the option of hosting parts of the 
RI on Windows Azure and be able to illustrate how you can use CQRS for 
cloud-hosted applications. 

* **Large.** We wanted to be able to show how our domain can be broken 
down into multiple bounded contexts to highlight when to use and when 
not use CQRS. We also wanted to illustrate how multiple architectural 
approaches (CQRS, CQRS/ES, CRUD) and legacy systems can co-exist 
within the same domain. We also wanted to show how multiple 
development teams could carry out work in parallel. 

* **Easily deployable.** The RI needed to be easily deployable so that you 
can install it and experiment with it as you read this guidance. 

As a result, we chose to implement the conference management system that 
Chapter 1, The Contoso Conference Management System introduces. 

[fig1]:           images/Map.png?raw=true