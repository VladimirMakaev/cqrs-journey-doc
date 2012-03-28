## Chapter 2
#Decomposing the Domain 

*Planning the journey: What will be the main stopping places along the way?*

#Definitions Used in This Chapter 

* Working definitions for this chapter 
* Domain 
* Bounded Context 

#Bounded Contexts in the Conference Management System 

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
defined by the **Conference Owner**) during which the registrant can 
complete the ordering process by making a payment for those seats. If 
the registrant does not pay for the tickets within _n_ minutes, the 
system cancels the **Reservation** and the seats become available to 
other registrants to reserve. 

## The Registrations Bounded Context

#Why Did We Choose These Bounded Contexts? 
