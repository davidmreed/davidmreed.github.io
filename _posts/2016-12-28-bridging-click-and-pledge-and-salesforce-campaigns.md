---
layout: post
title: Bridging Click & Pledge and Salesforce Campaigns with Process and Flow
---

Click & Pledge provides functionality to add contacts who register for an event to a Salesforce campaign. However, contacts are added with the first available status (often 'Sent'), rather than a value showing them as registered or attended. Because campaigns provide a standard, deduplicated, reportable data architecture that's independent of registration package, it's valuable to propagate C&P status updates into the associated campaigns. At the same time, it's easy to meet a common campaign reporting need by marking who's attending each event as a guest of whom, using a custom lookup field "Registered by" to the `Contact` on the `Campaign Member` object.

Because both of the registration objects used by Click and Pledge (`C&P Event Registered Attendee` and `C&P Event Registrant`) may be associated with `C&P Temporary Contacts` that can be linked to `Contacts` in any order, at different times, and via asynchronous process, we use an adaptable structure with two processes and two autolaunched flows. The processes and flows create and update `Campaign Members` incrementally as information becomes available, and re-run on each modification to the registration objects to propagate updates.

## Process 1: Registered Attendees
Triggered on the `C&P Event Registered Attendee` object upon creation or modification.

The single action group is run when the following formula evaluates to `true`:

    !ISBLANK([CnP_PaaS_EVT__Event_attendee_session__c].CnP_PaaS_EVT__ContactId__c) && (!ISBLANK([CnP_PaaS_EVT__Event_attendee_session__c].CnP_PaaS_EVT__Registration_level__c) && !ISBLANK([CnP_PaaS_EVT__Event_attendee_session__c].CnP_PaaS_EVT__Registration_level__c.CnP_PaaS_EVT__Campaign__c))

The formula ensures that the process is only invoked when both a contact is assigned (i.e., after the `C&P Temporary Contact` is processed) and a campaign is assigned for the relevant registration level.

The only action is Run Flow, triggering Flow 1 with the Registered Attendee and Campaign IDs as parameters.

![Run Flow]({{ site.baseurl }}/public/cnp-campaign-screens/process1.png)

## Process 2: Registrants
Triggered on the `C&P Event Registrant` object upon creation or modification. This process is required only for populating the "Registered by" field on the `Campaign Member`.

The sole action group criterion is `[CnP_PaaS_EVT__Event_registrant_session__c].CnP_PaaS_EVT__ContactId__c` is not `null`.

The only action is Run Flow, triggering Flow 2 with the Registrant ID as a parameter.

## Flow 1: Update Campaign Members
*Called by Process 1 and Flow 2*

This flow contains the meat of the functionality. It accepts two input variables, holding the IDs of the `C&P Event Registered Attendee` and the associated `Campaign`. The flow has the following structure.

![Update Campaign Members flow]({{ site.baseurl }}/public/cnp-campaign-screens/flow1.png)

Three Fast Lookup elements assign the `C&P Event Registered Attendee`, `Campaign Member` (if extant), and `C&P Event Registrant` (if extant) objects to variables. Note that the registered attendee is guaranteed not to be `null` by the calling process, but the registrant is not — its presence depends on how each individual transaction is processed by the user.

If the `Campaign Member` is `null`, a new record is created and inserted. If not, the `Campaign Member` is updated. In either case, the status is set to "Registered" or "Attended" depending on the value of the `Check-In Status` field. If the registrant has a `Contact` assigned, the "Registered by" lookup is populated; if not, Process 2 will fire when it's updated to bring the change through.

## Flow 2: Update Registered Attendees
*Called by Process 2, calls Flow 1*

This flow is required only for populating the "Registered by" field on the `Campaign Member` in the circumstance that the `C&P Temporary Contact` corresponding to the registrant is processed after one or more of those corresponding to the associated attendees. It accepts as input parameter the ID of the modified `C&P Event Registrant` object. The flow has the following structure.

![Update Registered Attendees flow]({{ site.baseurl }}/public/cnp-campaign-screens/flow2.png)

A Fast Lookup assigns all `C&P Registered Attendees` linked to the registrant to an sObject collection variable. A loop iterates over the collection and checks for a valid registration level with a campaign value populated. If it finds those values, it invokes Flow 1 with the required parameters. Any other registered attendees are ignored.

## Summary

Unifying information about event registrations with campaigns enhances reportability, particularly in a context where multiple registration packages or types of event are in use (Click and Pledge, Google Forms, small events without online registrations). Processes and Flows offer an easy way to migrate Click and Pledge registration information to campaigns in real time.
