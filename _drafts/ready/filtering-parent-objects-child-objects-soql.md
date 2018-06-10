---
layout: post
title: "Everyday Salesforce Patterns: Filtering Parent Objects By Child Objects"
---

Sometimes, we need to filter an `Account` query by its `Contacts`, or some custom object `Project__c` by its associated `Subject_Area__c` records. There might not be rollup summary fields in place, or the criteria might go beyond what rollups can do, or we might be dealing with a lookup relationship, forcing us to express this filtration directly in SOQL or in Apex. 

Suppose, for example, that we do have a custom object `Project__c`. A second custom object `Subject_Area__c` is connected to it by a lookup relationship, `Linked_Project__c`, with the relationship name `Subject_Areas`. Suppose further that `Subject_Area__c` also has a junction object `Subject_Area_Expert__c` creating a many-to-many relationship to Contact; `Subject_Area_Expert__c` also records the dates an expert is assigned to that subject area.

We'd like to be able to a perform a variety of queries that express different kinds of filters on the parent based upon characteristics and qualities of the different child objects:

 1. Locating `Project__c` records with no subject areas at all.
 1. Locating `Project__c` records with more than five subject areas.
 1. Locating `Project__c` records with subject areas whose name contains 'Industry'.
 1. Locating `Project__c` records without subject areas whose name contains 'Industry'.
 1. Locating `Project__c` records with at least one `Subject_Area__c` record that itself has at least one `Subject_Area_Expert__c` assignment with current dates to a Contact whose Title is "Solution Architect".

 We should also note that some of these requirements can potentially be met using native Salesforce reports with Cross Filters, or by the use of Declarative Lookup Rollup Summaries.

 There are several basic ways to approach constructing these queries: 

- Using a parent-child query and performing filtering in Apex.
- Using a parent query with an `IN` or `NOT IN` child subquery with filter (semi-join/anti-join).
- Using a child aggregate query and postprocessing in Apex.
- Linking multiple queries of these kinds by performing a synthetic join in Apex.

## Parent-Child Query

    for (Project__c sr : [SELECT Id, 
                                 (SELECT Id 
                                  FROM Subject_Areas__r 
                                  WHERE Name LIKE '%Industry%') 
                          FROM Project__c 
                          WHERE Client__c = :clientId]) {
        if (sr.Subject_Areas__r.size() == 0) {
            // We've identified a Project on the client whose Id is `clientId` with no Subject Areas that contain 'Industry'.
            // (but potentially other Subject Areas)
            // Do something about it.
        }
    }

This pattern is appropriate for use when the filtration criteria for child objects are very complex or involve relationships that cannot be easily expressed in SOQL, and for situations where we are looking for a zero count of child objects. Note that the `WHERE` filters on the parent and child query are, strictly speaking, optional, although performance in many cases will necessitate selective query filters. It is a simple pattern and can implement requirements 1-4, but not requirement 5, as only one level of child relationship can be traversed.

This can be an anti-pattern for situations with high data volume in the parent or child object, when query performance will be a huge concern and heap size could become an issue. Adding selective filters on the parent object, adding filters on the child object, and reducing the number of columns queried will help obviate these issues.

While parent-child queries can only descend one level of relationship in the object hierarchy, Apex post-processing can gather child object Ids and re-query to descend additional plies.

## Parent Query with `IN` or `NOT IN` Child Subquery (Semi-Join/Anti-Join)

Parent queries with `IN` and `NOT IN` child subqueries are particularly useful for cases 3 and 4. This query format takes advantage of the fact that the subquery is treated as returning a typed `Id` value, not an sObject instance. If we ran the subquery below, on `Subject_Area__c`, separately in Apex, we'd get back a `List<Subject_Area__c>`, which we'd have to iterate over and accumulate parent `Project__c` Ids before re-querying that object. When we present it as a subquery, no intermediate steps or type conversion are required; the Ids are used directly. (Salesforce does require that they be Ids of the correct type of object, however).

    SELECT Id 
    FROM Project__c 
    WHERE Id NOT IN (SELECT Linked_Project__c 
                     FROM Subject_Area__c 
                     WHERE Name LIKE '%Industry%')

See [Semi Joins and Anti-Joins](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql_select_comparisonoperators.htm#semijoin_and_antijoin) for more in-depth information, including the numerous restrictions that apply to this type of query.

This pattern is particularly suitable for case 3 and case 4, where we're looking for parent objects based on specific criteria on (but not volume of) child objects. Note that it can return parent object data but doesn't include child object data unless we include a separate sub-select. 

In particularly complex cases or if multiple levels of subquery are required, it's necessary to run the subqueries separately in Apex and accumulate relevant Ids in a `Set` for inclusion in the next query layer. Salesforce only allows one level of semi-join or anti-join, so we perform a sort of synthetic join in Apex of two separately expressed SOQL queries. For example, we can cover case 5 in a fashion like this:

    // 5. Locating `Project__c` records with at least one `Subject_Area__c` record that itself has at least one `Subject_Area_Expert__c` assignment with current dates to a Contact whose Title is "Solution Architect".

    List<Subject_Area_Expert__c> experts = [SELECT Subject_Area__c
                                            FROM Subject_Area_Expert__c
                                            WHERE Contact__c.Title = 'Solution Architect'
                                                  AND Start_Date__c <= TODAY
                                                  AND End_Date__c >= TODAY];
    Set<Id> subjectAreaIds = new Set<Id>();

    for (Subject_Area_Expert__c e : experts) {
        subjectAreaIds.add(e.Subject_Area__c);
    }
    
    List<Project__c> = [SELECT Id 
                        FROM Project__c 
                        WHERE Id IN (SELECT Linked_Project__c 
                                     FROM Subject_Area__c 
                                     WHERE Id IN :subjectIds)];

The same Apex/SOQL pattern can be extended to cover a more or less arbitrary depth of complexity, up to the point where governor limits are implicated.

## Child Aggregate Query

The child aggregate query is suitable for locating (and ordering) parent records based on a non-zero count of child records, optionally matching the child records against some criterion. As such, it can cover our Cases 2 and 3 well. It's not suitable for any situation where parent records without children should be included. It can provide a count of child records matching specific criteria, but doesn't return the child or parent record data itself. The query can express some filtration that would otherwise need to be performed in Apex.

    SELECT count(Id) count, Linked_Project__c 
    FROM Subject_Area__c 
    WHERE Name LIKE '%Industry%' 
    GROUP BY Linked_Project__c 
    HAVING count(Id) > 2 
    ORDER BY count(Id) DESC

This query will give us a `List<AggregateResult>` with the Ids of every `Project__c` having at least two associated `Subject_Area__c` records whose Names contain 'Industry', in descending order of the count of such records. We'll have to re-query to get more information about the Projects themselves or otherwise post-process the `AggregateResult` list in Apex.

In large data volume environments, selectivity may be a challenge because the filters on the child object that are relevant to the desired parent object set may not be especially narrow relative to the overall child object data set. Testing and query planning will help pinpoint any performance issues.