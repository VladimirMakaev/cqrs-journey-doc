## Chapter 2
# Decomposing the Domain 

*Planning the journey: What will be the main stopping places along the way?*

# Definitions Used in This Chapter 

The following definitions are used for the remainder of this chapter. 
For more detail, and possible alternative definitions see [CQRS in 
Context][r_chapter1] in the Reference Guide. 

## Domain 

The **Domain** refers to the business domain for the Contoso Conference 
Management System (the reference implmentation). The previous chapter 
[The Contoso Conference Management System][j_chapter1] provides an 
overview of this domain. 

## Bounded Context 

The term **Bounded Context** comes from the book "Domain-Driven Design: 
Tackling Complexity in the Heart of Software" by Eric Evans. In brief, 
this concept is introduced as a way to decompose a large, complex system 
into more manageable pieces: a large system is composed of multiple 
bounded contexts. Each bounded context is the context for its own 
self-contained domain model, and has its own ubiquitous language. You 
can also view a bounded context as autonomous business component 
defining clear consistency boundaries: one bounded context may only 
communicate with another bounded context by raising events. 

# Bounded Contexts in the Conference Management System 

## The Orders and Reservations Bounded Context

When a registrant interacts with the system, the system creates an 
**Order** to manage the **Reservations**, payment, and 
**Registrations**. An Order contains one of more Order Items. 

A **Reservation** is a temporary reservation of one or more seats at 
conference. When a registrant begins the ordering process to purchase a 
number of seats at conference, the system creates **Reservations** for 
the number of seats requested by the registrant. These seats are then 
not available for other registrants to reserve. The **Reservations** are 
held for _n_ minutes (_n_ is one of the properties of the conference 
defined by the **Business Customer**) during which the registrant can 
complete the ordering process by making a payment for those seats. If 
the registrant does not pay for the tickets within _n_ minutes, the 
system deletes the **Reservation** and the seats become available to 
other registrants to reserve. 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	At the moment the value of _n_ is hardcoded - this may change in the future and become a parameter than can be set by the Business Customer.
  </span>
</div>

## The Registrations Bounded Context

## The Wait-listing Bounded Context

## The Payments Bounded Context 

# Why Did We Choose These Bounded Contexts? 


[j_chapter1]:	  Journey_01_Introduction.markdown
[r_chapter1]:     Reference_01_CQRSInContext.markdown