---
layout: post
title: Everyday Salesforce Patterns: Filtering Parent Objects By Child Objects 
---

Sometimes, we need to filter an `Account` query by its `Contacts`, or some custom object `Project__c` by its associated `Subject_Area__c` records. There might not be rollup summary fields in place, or the criteria might go beyond what rollups can do, or we might be dealing with a lookup relationship. 

Suppose, for example, that we do have a custom object `Project__c`. A second custom object `Subject_Area__c` is connected to it by a lookup relationship, `Linked_Project__c`, with the relationship name `Subject_Areas`. We'd like to be able to handle several query requirements:

 1. Locating `Project__c` records with no subject areas at all.
 1. Locating `Project__c` records with more than five subject areas.
 1. Locating `Project__c` records with subject areas whose name contains 'Industry'.

 There are three basic ways to approach constructing these queries: 

 - Using a parent-child query and performing filtering in Apex.
 - Using a parent query with an `IN` child subquery with filter.
 - Using a child aggregate query.

## Parent-Child Query Filtered in Apex

    for (Project__c sr : [SELECT Id, (SELECT Id FROM Subject_Areas__r) FROM Project__c]) {
        if (sr.Subject_Areas__r.size() == 0) {
            // We've identified a Project without any Subject Areas.
            // Do something about it.
        }
    }

This pattern is appropriate for use when the filtration criteria for child objects are very complex or involve relationships that cannot be easily expressed in SOQL, and for situations where we are looking for the absence of child objects. It is a simple pattern and can implement all three of our requirements, although it may not be the most elegant or performant solution for cases 2 and 3.

This can be an anti-pattern for situations with high data volume in the parent or child object, when query performance will be a huge concern and heap size could become an issue. Adding selective filters on the parent object in the SOQL query will help obviate this concern; adding filters on the child object will also address issues caused by heap size constraints if child volume is substantial.

## Parent Query with `IN` or `NOT IN` Child Subquery

This query format takes advantage of the fact that `Id` columns in subqueries aren't typed. If we ran the subquery separately in Apex, we'd get back a `List<Subject_Area__c>`, which we'd have to iterate over and accumulate the parent Ids before re-querying. When we present it as a subquery, no intermediate steps or type conversion are required; the Ids are used directly.

    SELECT Id FROM Project__c WHERE Id NOT IN (SELECT Linked_Project__c FROM Subject_Area__c WHERE Name LIKE '%Industry%')

This pattern is particularly suitable for case 3, where we're looking for parent objects based on specific criteria (but not volume) of child objects. Note that it can return parent object data but doesn't include child object data unless we include a separate sub-select.

In particularly complex cases or if multiple levels of subquery are required, it may be necessary to run the subqueries separately in Apex and accumulate relevant Ids in a `Set` for inclusion in the next query layer. For example, the above query could be factored in Apex:

    List<Subject_Area__c> areas = [SELECT Linked_Project__c FROM Subject_Area__c WHERE Name LIKE '%Industry%'];
    Set<Id> projectIds = new Set<Id>();

    for (Subject_Area__c s : areas) {
        projectIds.add(s.Linked_Project__c);
    }

    List<Project__c> = [SELECT Id FROM Project__c WHERE Id NOT IN :projectIds];

The same Apex/SOQL pattern can be extended to cover multiple levels of subquery. 

Since the `Id` field is always indexed, the overall performance should be roughly equivalent to the total individual performance of the linked queries.

## Child Aggregate Query

The child aggregate query is suitable for locating (and ordering) parent records based on a non-zero count of child records, optionally matching the child records against some criterion. It's not suitable for any situation where parent records without children should be included. It can provide a count of child records matching specific criteria, but doesn't return the child or parent record data itself.

    SELECT count(Id), Linked_Project__c FROM Subject_Area__c WHERE Name LIKE '%Industry%' GROUP BY Linked_Project__c HAVING count(Id) > 2 ORDER BY count(Id) DESC

This query will give us a `List<AggregateResult>` with the Ids of every `Project__c` having at least two associated `Subject_Area__c` records whose Names contain 'Industry', in descending order of the count of such records. We'll likely have to re-query to get more information about the Projects themselves or otherwise post-process the `AggregateResult` list in Apex.

It can be difficult to make queries of this kind sufficiently selective, since the filters on the child object that are relevant to the desired parent object set may not be especially narrow relative to the overall child object data set. Testing and query planning will help pinpoint any performance issues.