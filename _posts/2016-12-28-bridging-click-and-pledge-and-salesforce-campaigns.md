---
layout: post
title: Bridging Click & Pledge and Salesforce Campaigns with Process and Flow
---

*This post has been extensively revised to broaden the applicability of the solution, covering anonymous and named Click & Pledge events.*

Click & Pledge provides functionality to add contacts who register for an event to a Salesforce campaign. However, contacts are added with the first available status (often 'Sent'), rather than a value showing them as registered or attended. Because campaigns provide a standard, deduplicated, reportable data architecture that's independent of registration package, it's valuable to propagate C&P status updates into the associated campaigns.

At the same time, we can add value to the `Campaign Member` object and support common reporting needs by marking who's attending each event as a guest of whom, using a custom lookup field `Registered by` to the `Contact` on the `Campaign Member` object.

Because both of the registration objects used by Click and Pledge (`C&P Event Registered Attendee` and `C&P Event Registrant`) may be associated with `C&P Temporary Contacts` that can be linked to `Contacts` in any order, at different times, and via asynchronous process, we use an adaptable structure with two processes and two autolaunched flows. The processes and flows create and update `Campaign Members` incrementally as information becomes available, and re-run on each modification to the registration objects to propagate updates.

It's also important to take into account anonymous events. Click & Pledge creates `C&P Registered Attendee` records for anonymous registrations, but they're never linked to contacts — only the `C&P Event Registrant` is. Our flows take this variation into account.

## Process 1: Registered Attendees
Triggered on the `C&P Event Registered Attendee` object upon creation or modification.

Because we cannot rely on a Contact ever being linked to the registered attendee (in the case of anonymous registrations),
the process's single action group is run without criteria. It simply invokes Flow 1 with the Registered Attendee as its parameter. The flow is responsible for extracting the other required information from the object hierarchy.

## Process 2: Registrants
Triggered on the `C&P Event Registrant` object upon creation or modification.

Even in the case of anonymous events, a Contact must ultimately be assigned to the registrant. This process therefore has a single action group criterion, `[CnP_PaaS_EVT__Event_registrant_session__c].CnP_PaaS_EVT__ContactId__c is not null`.

The only action is Run Flow, triggering Flow 2 with the Registrant ID as a parameter. This process is necessary not only to update the `Registered by` field, but also to handle propagating a Contact value down to its registered attendees for anonymous events.

## Flow 1: Update Campaign Members
*Called by Process 1 and Flow 2*

This flow contains the meat of the functionality. It accepts one input variable, holding the ID of the `C&P Event Registered Attendee`. The flow has the following structure.

![Update Campaign Members flow]({{ site.baseurl }}/public/cnp-campaign-screens/flow1.png)

Four Fast Lookup elements assign the `C&P Event Registered Attendee`, `C&P Event`, `Campaign Member`, and `C&P Event Registrant` objects to variables. Note that the registered attendee is guaranteed not to be `null` by the calling process, but the other objects are nullable, and the registrant will often be `null` dependent upon the user's processing of temporary contacts.

The flow uses five formulas to draw data from these four objects:

### `CampaignId`

Evaluates to the Campaign assigned to the registration level, if any, or to that assigned to the event, if any, or `null`.

   IF(!ISBLANK({!RegisteredAttendeeSobject.CnP_PaaS_EVT__Registration_level__c}) && !ISBLANK({!RegistrationLevelSobject.CnP_PaaS_EVT__Campaign__c}), {!RegistrationLevelSobject.CnP_PaaS_EVT__Campaign__c},
   IF(!ISBLANK({!RegisteredAttendeeSobject.CnP_PaaS_EVT__EventId__c}) && !ISBLANK({!EventSobject.CnP_PaaS_EVT__Campaign__c}), {!EventSobject.CnP_PaaS_EVT__Campaign__c}, null))

### `ContactId`

Evaluates to the Contact assigned to the registered attendee, unless the event is marked as anonymous, in which case it evaluates
to the contact of the registrant.

   IF(!ISBLANK({!RegisteredAttendeeSobject.CnP_PaaS_EVT__EventId__c}) && {!EventSobject.CnP_PaaS_EVT__Anonymous__c} && !ISBLANK({!RegisteredAttendeeSobject.CnP_PaaS_EVT__Registrant_session_Id__c}) && !ISBLANK({!Registrant.CnP_PaaS_EVT__ContactId__c}), {!Registrant.CnP_PaaS_EVT__ContactId__c}, {!RegisteredAttendeeSobject.CnP_PaaS_EVT__ContactId__c})

### `RegistrantId`

Evaluates to the Contact assigned to the Registrant, unless it is the same as the Contact of the attendee. Used to populate the `Registered by` field.

   IF(!ISBLANK({!RegisteredAttendeeSobject.CnP_PaaS_EVT__Registrant_session_Id__c}) && !ISBLANK({!Registrant.CnP_PaaS_EVT__ContactId__c}) && {!Registrant.CnP_PaaS_EVT__ContactId__c} != {!ContactId}, {!Registrant.CnP_PaaS_EVT__ContactId__c}, null)


### `CMStatusExisting`

Evaluates to the campaign member status value to use if an existing `Campaign Member` is located. It will not overwrite an 'Attended' value; this attempts to cope with check-ins performed on the multiple registered attendees that may be linked to a single contact and registrant for anonymous events.

   IF({!RegisteredAttendeeSobject.CnP_PaaS_EVT__CheckIn_Status__c} = 'Checked-In', 'Attended', IF(!ISPICKVAL({!CM.Status}, 'Attended'), 'Registered', TEXT({!CM.Status})))

### `CMStatusNew`

Evaluates to the campaign member status value to use if a new `Campaign Member` is being created.

   IF({!RegisteredAttendeeSobject.CnP_PaaS_EVT__CheckIn_Status__c} = 'Checked-In', 'Attended', 'Registered')


Because the calling processes cannot guarantee that appropriate information will be available in the object hierarchy in all
circumstances, the flow uses a decision element to decide whether to proceed; it continues if both `ContactId` and `CampaignId` are non-null.

These conditions being met, if the `Campaign Member` is `null`, a new record is created and inserted, using `CMStatusNew`, `ContactId`, `CampaignId`, and `RegistrantId`. If not, the `Campaign Member` is updated, and `CMStatusExisting` is used.

If the registrant has not yet been populated (for a named event), the `Registered by` field will be populated when Process 2 and Flow 2 fire.

## Flow 2: Update Registered Attendees
*Called by Process 2, calls Flow 1*

This flow is required for handling anonymous registrations (where a contact is assigned only to the registrant object) and for populating the "Registered by" field on the `Campaign Member` in the circumstance that the `C&P Temporary Contact` corresponding to the registrant is processed after one or more of those corresponding to the associated attendees. It accepts as input parameter the ID of the modified `C&P Event Registrant` object. The flow has the following structure.

![Update Registered Attendees flow]({{ site.baseurl }}/public/cnp-campaign-screens/flow2.png)

A Fast Lookup assigns all `C&P Registered Attendees` linked to the registrant to an sObject collection variable. A loop iterates over the collection and invokes Flow 1 with the required parameter. No criteria are applied to the registered attendees; all evaluation is handled in Flow 1.

## Summary

Unifying information about event registrations with campaigns enhances reportability, particularly in a context where multiple registration packages or types of event are in use (Click and Pledge, Google Forms, small events without online registrations). Processes and Flows offer an easy way to migrate Click and Pledge registration information to campaigns in real time.
