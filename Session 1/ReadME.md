# Vacation-Tracking-System (VTS)

This ReadMe file contains technical analysis, system diagrams(Flow Chart, Class Diagram, Sequence Flow Diagram), and documentation for a **Vacation Tracking System (VTS)**.

---

## Vision

To provide individual employees with the capability to manage their own vacation time, sick leave, and personal time off, without having to be an expert 
in company policy or the local facility's leave policies. This will streamline the functions of HR department, to minimize noncore, business-related activities
of management. 

The system must by easy to use.

---

## Functional Requirements
The system will provide the following key features:

+ Implement a flexible rule-based system for validating and verifying leave time requests.  
+ Enabele manager approval (optional).  
+ Provide access to requests for the previous calendar year and allows requests to be made up to a year and half in the future.  
+ Uses e-mail notification to request manager approval and notify employees.
+ Request manager approval.  
+ Notify employees of request status changes.  
+ Enables the HR and system administration personnel to override all actions restricted by rules, with logging of those overrides 
+ Allows managers to directly award personal leave time (with system-set limits)
+ Provides a Web service interface for other internal systems to query any given employee’s vacation request summary
+ The employee should be able to:
  + View vacation time requests.  
  + Create new requests.  
  + Cancel existing vacation time requests.


## Non-Functional Requirements

+ The system should be easy to use  
+ The system should be implemented as an extension to the existing intranet portal system** and use the portal’s single sign-on (SSO) for authentication.  
+ The system should be a web application, extending the organization’s existing intranet to provide a convenient and natural entry point for users.

---

## Constraints

+ The system must be accessible through the internal network only.
+ The system must be compatible with existing hardware.

  
## Assumptions

The **Vacation Tracking System (VTS)** is not expected to be a heavily used application.  

The system will also be integrated with other internal applications, the following considerations should be applied:

+ The system should be designed for easy integrationwith any other internal system.  
+ The system should be capable of handling approximately 300–500 concurrent requests, which will be sufficient for the expected load.


## Actors

### 1. Employee
The main user of the system.  
An employee uses VTS to manage their vacation time, including viewing, creating, and canceling vacation requests.

### 2. Manager
A manager has all the capabilities of a regular employee but also:  
+ Approves vacation requests from direct subordinates.  
+ Can award subordinates compensatory (comp) time within system-defined limits.

### 3. System Administrator
Responsible for:  
- The operation of system technical resources (e.g., web server, database).  
- Collecting and archiving system log files.

**All System Diagrams are found in the Diagrams folder**
