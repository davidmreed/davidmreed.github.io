---
layout: post
title: "Nested Iteration: A Quick Look at Computational Complexity in Apex"
---

The logic of nested iteration can be a real trap for new programmers. I see this a lot on [Salesforce Stack Exchange](https://salesforce.stackexchange.com), coming from two different angles. One angle simply asserts, having been taught so, that "nested loops are bad." Well, not exactly - not as such, although they *can* be in specific implementations. The other side perceives no danger at all in nested loops - indeed, finds them the most natural, if naive, route to expressing certain search constructs.

Both miss the mark through not understanding the notion of *computational complexity* and how to evaluate it in programmatic logic. While it's important to programmers everywhere, computational complexity can be especially important for developers on the Salesforce platform, where governor limits bite hard on reckless use of resources like CPU time.

I want to look at a simple but broadly applicable example of a "bad" nested iteration, one that needlessly consumes CPU time through inefficient logic, and the patterns by which one fixes that pattern. Secondly, I'd like to look at an example of nested iteration that's quite correct, and needed for its intended purpose.

## Updating Opportunities Based on Account Changes

Let's start with an Account trigger. Our requirement is as follows:

> As a Sales representative, I need the Description field on each Opportunity to match that on the parent Account so that I always have access to critical Account information while working deals.

We'll suppose that using a custom formula field is contraindicated, for the sake of illustration. We prepare the following trigger:

```apex
trigger AccountTrigger on Account (after update) {
    List<Opportunity> opps = [
        SELECT Id, Description
        FROM Opportunity
        WHERE AccountId IN :Trigger.newMap.keySet()
    ];
    Map<Id, Opportunity> updateMap = new Map<Id, Opportunity>();

    for (Account a : Trigger.new) {
        for (Opportunity o : opps) {
            if (o.AccountId == a.Id) {
                o.Description = a.Description;
                updateMap.put(o.Id, o);
            }
        }
    }

    update updateMap.values();
}
```

This trigger does some things right. In particular, it's bulkified, costing us only one SOQL query and one DML statement. However, the structure of the nested `for` loops puts us at risk.

I'm calling this pattern a "matrix search" - a term I got from [Andrés Catalán](https://twitter.com/sfdx__andres/status/1191298893662838785) - because I like how it evokes scanning across rows and columns of data. It's a pattern that can be rather nasty where it's not needed, and it eats CPU time. Let's look at why.

The key question to ask here is "How many times will the body of the inner loop be executed?" Here, how many times will we ask `if (o.AccountId == a.Id)`?

Suppose we update 20 Accounts, and those Accounts have an average of 15 Opportunities, for a total of 300 Opportunities returned to our SOQL query. Our outer loop will execute 20 times (once per Account). Our inner loop will execute 300 times *per Account*, because we're iterating over all of those Opportunities - even the ones that are totally irrelevant to the Account we're looking at.

So what's the total invocation count of `if (o.AccountId == a.Id)`? It's the total number of Accounts (20) multiplied by the total number of Opportunities (300): 6,000 iterations, to process just 300 records.

Imagine if you're in a large-data-volume org, or if you're performing updates via Data Loader. These numbers can get much, much worse. 200 Accounts updated via Data Loader, with an average of 20 Opportunities each (total of 4,000)? Suddenly, we're looking at *800,000* iterations of that loop. We'll be in CPU limit trouble before we know it, and our users will notice the performance hit well before that point. We've needlessly increased our computational complexity.

> Οἰόνται τινές, βασιλεῦ Γέλων, τοῦ ψάμμου τὸν ἀριθμὸν ἄπειρον εἶμεν τῷ
πλήθει <br/>
Some believe, King Gelon, that the number of the sand - in regard to its multitude - is without limit...<br />
— Archimedes, *The Sand Reckoner*


### The Fix: Maps

Nearly any time we're iterating over two lists in nested `for` loops and making comparisons between every possible pair of values, which is exactly what we're doing here, we can eliminate the inner loop using a `Map` for significant performance and complexity gains.

We're interested in an equality between two values, `Account.Id` and `Opportunity.AccountId`. For each `AccountId`, we need to find the corresponding Account - fast - and access other data on the Account. A `Map<Id, Account>` lets us do that.

Instead of nesting iteration, we'd arrange for a `Map<Id, Account>` *before* we start iterating over our Opportunities. Then, inside the Opportunity loop, we don't need to iterate over Accounts to find the one with the matching Id, because we can access it directly via the Map's `get()` method, with no iteration - in *constant time*, that doesn't get more expensive with the number of Accounts we have.

In this use case, we don't need to build the Map ourselves, because we've already got `Trigger.newMap`, which here is a `Map<Id, Account>`. 

If we needed to build the Map, or we were working with some other property than the Id, like the Name, we'd do

```apex
Map<String, Account> accountMap = new Map<String, Account>();

for (Account a : accountList) {
    accountMap.put(a.Name, a); 
}
```

However we obtain our Map, with it in hand, we iterate over Opportunities - once, and with no nested iteration.

```apex
for (Opportunity o : opps) {
    Account a = accountMap.get(o.AccountId); // No iteration!
    o.Description = a.Description;
    updateMap.put(o.Id, o);
}
```

Note that the first loop, on Account, will execute exactly once per Account. Then, the second loop on Opportunity will execute exactly once per Opportunity. Taking the higher-data example from above, we go from

    200 Accounts * 4,000 total Opportunities = 800,000 iterations

to 

    200 Accounts + 4,000 total Opportunities = 4,200 iterations

a reduction of roughly *99.5%*.

This pattern is highly generalizable, and it's why it's so critical to ask those questions about how many times your code will execute given various data inputs. 

In computer science, these analyses are typically done in terms of Big-O notation. We won't go into the details of Big-O notation here, but keep the phrase in the back of your mind - it's a mathematical way to formalize the cost of specific algorithms.

> Note: there are at least two potential performance optimizations in the final version of the trigger given above. Both optimizations reduce the scope of the operations to be performed. Finding them is left as an exercise for the reader.

## Counter-Example: Iterating Over SOQL Parent-Child Query

Let's look at another example where nested iteration isn't pathological - and in fact is the correct implementation pattern. Here's a new user story:

> As a Customer Support supervisor, I need to mark my Accounts' Status field "Critical" whenever they have more than 10 open Cases matching a complex set of criteria.

There's a few different ways to implement this requirement. We'll look at a fragment of one: we develop a service class that takes a set of Account Ids against whose Accounts new Cases have been opened. Because the Case criteria are so complex, we're using a pre-existing utility class to evaluate each Case to see whether it matches, and we're running a parent-child SOQL query to walk through available Cases on each Account.

```apex
public static void evaluatePotentialCriticalAccounts(Set<Id> accountIds) {
    List<Account> updateList = new List<Account>();

    for (Account a : [
        SELECT Id, Status__c,
               (SELECT Id, Subject, Status FROM Cases)
        FROM Account
        WHERE Id IN :accountIds
    ]) {
        Integer qualifyingCases = 0;

        for (Case c : a.Cases) {
            if (CaseUtility.isCaseCriticalStatusEligible(c)) {
                qualifyingCases++;
            }
        }

        if (qualifyingCases >= 10) {
            updateList.add(new Account(Id = a.Id, Status__c = 'Critical'));
        }
    }

    update updateList;
}
```

Now, this code might not be ideal (depending, of course, on the fine detail of our requirements and the configuration of our Salesforce org). For example, if we can fit our criteria into a SOQL query instead of an Apex function, an aggregate query against Case would likely be a better solution than this Apex.

However, what this code _doesn't_ do is fall into the same trap as the trigger we looked at above, even though it uses nested loops. Why? 

Look at how many times the loops run. We'll call `CaseUtility` exactly once per Case - *not* the total number of Cases multiplied by the total number of Accounts. We'll never run a loop iteration that's irrelevant, that has no possibility of yielding action, because we're not "matrixing" our Accounts and Cases and generating data most of which is irrelevant to the work we're aiming to do. Hence, we hit each Case exactly once. There's no optimization possible via Maps here.

There are other counter-examples where nested iteration is perfectly valid. Consider for example a board-game algorithm. We might need to iterate over each space on the board, across rows and down columns. A nested `for` loop is just right for this use case, and it wastes no CPU time because traversing the entire two-dimensional space is exactly what is called for. 

In both situations, the computational complexity of the solution we've implemented is in line with the problem we're solving.

## Summing Up

Here's a few points I hope you'll take away from this discussion, whether or not you ever encounter the term *Big-O notation* again:

- Iteration, and nested iteration, are not bad in and of themselves.
- Traversing data you don't need to traverse _is_ bad.
- It's always worth asking "how many times will this run?"
- Performance can scale up and down much faster than you expect, given pathological implementations.