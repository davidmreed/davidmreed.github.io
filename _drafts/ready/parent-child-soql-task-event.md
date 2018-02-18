---
layout: post
title: "Everyday Salesforce Patterns: Child-Parent SOQL on Task and Event"
---

Performing child-parent SOQL is more complex than usual when the `Task` and `Event` objects are involved. That's because these objects include *polymorphic lookup fields*, `WhoId` and `WhatId`, which can point to any one of a number of different objects.

While a feature called [SOQL polymorphism](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql_relationships_and_polymorph_keys.htm) is in Developer Preview and would offer a SOQL-only way to obtain details about `Task` and `Event` parents in pure SOQL, unless and until it's made generally available, Apex is required to query parent details for these objects other than a tiny subset of Name-related fields. Here's an example of this pattern as it might be applied in a trigger. The key is the following steps:

 1. Iterating over the `Task` or `Event` records and accumulating the `WhatId` or `WhoId` values on a per-object basis;
 1. performing a single SOQL query per parent object type;
 1. creating a `Map<Id, sObject>` to allowing indexing the `WhoId`/`WhatId` into query results;
 1. and finally iterating a second time over the `Task` or `Event` records to perform work with the parent information available.

This sample implementation sets a checkbox field called `High_Priority__c` on the `Task` when its `WhatId` is either an open `Opportunity` or an `Account` whose `AnnualRevenue` is greater than one million dollars. Note that the pattern works the same way whether we're looking at `WhoId` or `WhatId`, and whether or not we're in a trigger context.

    trigger TaskTrigger on Task (before insert) {
        // In production, we would use a trigger framework; this is a simple example.
        
        // First, iterate over the set of Tasks and look at their WhatIds.
        // We can use the `What.Type` field to identify which parent object
        // it corresponds to, or cast the Id to a string and check the first three characters against the object's Key Prefix
        // We'll accumulate the WhatIds in Sets to query (1) for Account and (2) for Opportunity.
        
        Set<Id> accountIds, oppIds;
        
        accountIds = new Set<Id>();
        oppIds = new Set<Id>();
        
        for (Task t : Trigger.new) {
            if (String.isNotBlank(t.WhatId)) {
                if ((String)t.What.Type == 'Account') {
                    accountIds.add(t.WhatId);
                } else if (t.What.Type == 'Opportunity') {
                    oppIds.add(t.WhatId);
                }
            }
        }
        
        // We will query into Maps so that we can easily index into the parent with our WhatIds
        Map<Id, Account> acts;
        Map<Id, Opportunity> opps;
        
        // Now we can query for the parent objects.
        // Here, the parent object logic is entirely contained in the query;
        // it could also be implemented in the loop below.
        acts = new Map<Id, Account>([SELECT Id FROM Account WHERE Id IN :accountIds AND AnnualRevenue > 1000000]);
        opps = new Map<Id, Opportunity>([SELECT Id FROM Opportunity WHERE Id IN :oppIds AND IsClosed = false]);
        
        // We re-iterate over the Tasks in the trigger set and alter their fields based on the information
        // queried from their parents. Note that this is a before insert trigger so no DML is required.
        
        for (Task t : Trigger.new) {
            if (acts.containsKey(t.WhatId) || opps.containsKey(t.WhatId)) {
                // With more complex requirements, we could source data from the parent object
                // Rather than simply making a decision based upon it.
                
                t.High_Priority__c = true;
            }
        }
    }

This example of the pattern assumes we're starting from the `Task` and making some decision based on information in the parent. In other situations, we might query first for a set of Tasks in which we're interested (perhaps applying a filter on `WhatId` or `WhoId`), follow a similar pattern to source parent information, and then update the parent records - or a different object entirely. The skeleton of the solution, however, will remain the same.