---
layout: post
title: Rolling Up Opportunity Contact Roles with Change Data Capture and Async Apex Triggers
---

Salesforce admins, and perhaps especially nonprofit admins, have been wishing for a long time for the ability to build more functionality around Opportunity Contact Roles - like roll-up summary fields, validation rules, and triggers.

The new Change Data Capture offered intriguing possibilities because, unlike regular Apex triggers, the feature [supports change notifications for `OpportunityContactRole`](https://developer.salesforce.com/docs/atlas.en-us.change_data_capture.meta/change_data_capture/cdc_object_support.htm). With the new ability, in Summer '19, to [subscribe to Change Data Capture events in Async Apex Triggers](https://developer.salesforce.com/docs/atlas.en-us.change_data_capture.meta/change_data_capture/cdc_trigger_intro.htm), we now have a viable route to build completely on-platform features using these tools - and the first demo I wanted to build was a solution to roll up Opportunity Contact Roles.

If you want to drive straight into the code, find it [here](https://github.com/davidmreed/cdc-opportunitycontactrole-rollups) on GitHub.

## The Simple Route and Why It Doesn't Work

My first strategy was simply to call into [Declarative Lookup Rollup Summaries](https://github.com/afawcett/declarative-lookup-rollup-summaries) from an Async Apex trigger. That approach runs afoul, though, of one of the key differences between Async Triggers and standard Apex Triggers:

*Apex triggers operate on sObjects. Async Apex triggers operate on change events.*

Change events aren't sObjects, and don't contain a complete snapshot of the sObject record at a point in time. Rather, since the underlying Change Data Capture feature targets synchronization of *changes* to some persistent store, each event only contains enough information to apply a specific change to a stored record. 

The section [Apex Change Event Messages](https://developer.salesforce.com/docs/atlas.en-us.change_data_capture.meta/change_data_capture/cdc_event_fields_body.htm) in the Change Data Capture Developer Guide explains what data is actually available in each message:


> **Create**<br />
> For a new record, the event message contains all fields, whether populated or empty. [...]<br/>
> **Update**<br />
> For an updated record, the event message contains field values only for changed fields. Unchanged fields are present and empty (null), even if they contain a value in the record. [...]<br />
> **Delete**<br />
> For a deleted record, all record fields in the event message are empty (null).<br />
> **Undelete**<br />
>For an undeleted record, the event message contains all fields from the original record, including empty (null) fields and system fields.

For roll-up applications, the key is the `update` and `delete` events. Note that for `update`, we don't get *old* data, only new, and at the time we receive the event the records in the database have already been updated (we can't query for old values). Upon `delete`, we don't get the field data for the deleted sObjects at all! It's assumed that we, as consumers of this event stream, have some other persistent data store against which we're applying the ordered stream of change events.

To build a roll-up, we need to know the parent objects for deleted children, the parent objects for changed records, and the old parents for children that are reparented.

So Change Data Capture + Async Apex Triggers is a dead end for building rollups? Not at all. There's another strategy that allows us to take advantage of Change Data Capture's inherent facility for synchronizing data stores: we'll use a shadow table.

## Rolling Up a Shadow Table

*The SFDX project for everything we're going to demonstrate here is available on [GitHub](https://github.com/davidmreed/cdc-opportunitycontactrole-rollups).*

We can't roll up Opportunity Contact Roles to the Opportunity, but we easily can roll up some arbitrary Custom Object with a master-detail relationship. A shadow table, in which we mirror the Opportunity Contact Role object with a Custom Object, fits nicely as a solution: it both gives us the roll up we need without code, and provides us with an on-platform persistent data store against which Change Data Capture events can be applied as we create, update, and delete Opportunity Contact Role records.

Here's the schema we'll use: a custom object plus a native Rollup Summary Field on Opportunity.

  ![Shadow Object]({{ "/public/cdc-contactroles/shadow-table.png" | absolute_url }})

  ![Rollup Field]({{ "/public/cdc-contactroles/rollup.png" | absolute_url }})

Note the External Id field we've created to hold the Id of the corresponding Opportunity Contact Role. That's the linchpin of our synchronization effort, and we'll use it below to build an efficient sync trigger.

## Synchronizing the Data

To synchronize data from `OpportunityContactRole` into our shadow table, we'll use an Async Apex trigger processing the `OpportunityContactRoleChangeEvent` entity. First, we'll select the needed object in Setup under Change Data Capture:

  ![Change Data Capture Setup]({{ "/public/cdc-contactroles/selected-entities.png" | absolute_url }})

Then, we build a trigger. The code is [here](https://github.com/davidmreed/cdc-opportunitycontactrole-rollups/blob/master/force-app/main/default/triggers/OpportunityContactRoleChangeEventTrigger.trigger).

Our trigger's operation is different from what we'd see in typical, synchronous Apex. We iterate in order over the change events we receive and use them to build up two data sets. The first is a `Map<Id, Shadow_Opportunity_Contact_Role__c>`, which we use to store both new and updated shadow records derived from the change events. The keys, worth noting, are `OpportunityContactRole` Ids, ensuring that we create or update exactly one record for each `OpportunityContactRole` whose state incorporates all of the changes in our inbound event stream.

We also store a `Set<Id>` of the Ids of deleted `OpportunityContactRole` records. As we find delete events in our stream, we remove corresponding entries from our create and update `Map`, and add those Ids to the `Set`.

When we finish iterating through events - remember that this is an *ordered time stream* - these two data structures contain the union of all of the changes we need to apply to our shadow table. At that point, it's two simple DML statements to persist the changes:

    upsert createUpdateMap.values() Opportunity_Contact_Role_Id__c;
    delete [
        SELECT Id
        FROM Shadow_Opportunity_Contact_Role__c
        WHERE Opportunity_Contact_Role_Id__c IN :deleteIds
    ];

The index on `Opportunity_Contact_Role_Id__c` should keep these operations performant, and once they complete, the system updates our native Rollup Summary Field on the parent Opportunities.

## Testing

There's just a couple of extra wrinkles to testing Async Apex Triggers. We have a new method in the system `Test` class to enable the Change Data Capture feature, and it overrides system CDC settings to ensure that the code under test executes regardless of org settings:

    Test.enableChangeDataCapture();

Then, we require that CDC events are delivered and processed synchronously, using the tried-and-true `Test.startTest()` and `Test.stopTest()` calls, or by calling `Test.getEventBus().deliver()`.

Intermediate testing results, while working towards passage and full code coverage, can be tough to interpret: most logs are found under the `Automated Process` user rather than the context user, [requiring the use of trace flags](https://salesforce.stackexchange.com/questions/266283/how-do-i-see-debug-logs-for-change-data-capture-triggers-in-salesforce), but some (possibly those from `@testSetup` methods) do appear for the context user. Code coverage maps can also produce misleading results until full passage is achieved.

As part of the demo, I built a test class that achieves full coverage on the Async Apex Trigger. It's also part of the [GitHub project](https://github.com/davidmreed/cdc-opportunitycontactrole-rollups/blob/master/force-app/main/default/classes/OpportunityContactRoleChangeEvent_TEST.cls). (While the tests have good logic path coverage, they could stand to exercise bulk use cases better!)

## Results

The CDC + Async Apex Trigger solution doesn't add everything we might want with Opportunity Contact Roles. We still cannot write Validation Rules against the Opportunity that take Roles into account, because the rollup operation is run asynchronously, after the original transaction commits. It's also a near-real time, rather than real time, solution, so a brief delay may be perceptible before the rollup field updates. And lastly, because Async Apex Triggers run as the Automated Process user, Last Modified By fields won't show actual users' names once the rollup operation completes.

But what it does, it does well: we get all the functionality and performance of native roll-up summary fields against an object that's never supported that feature, with a minimal investment in code. Users can see, and report on, rolled-up totals of Opportunity Contact Roles across their Opportunity pipeline.

And it's a really neat way to apply some of the latest Salesforce technologies to solve real-world problems.