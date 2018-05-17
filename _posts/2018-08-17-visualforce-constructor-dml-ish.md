---
layout: post
title: DML-ish Operations in the Visualforce Controller Constructor 
---

DML is not allowed in a Visualforce page controller's constructor. This isn't news.

However, a colleague and I discovered that the behavior with various DML-equivalent ("DML-ish") code is inconsistent and rather strange.

Here's a minimal Visualforce page:

    <apex:page controller="TestBatchFiringController">
    </apex:page>

and its corresponding controller:

    public class TestBatchFiringController {
        public TestBatchFiringController() {
            // These statements successfully completed.

            // Results in a valid AsyncApexJob id, but batch does not execute.
            // AsyncApexJob Id is not valid after method executes (but it is in test context!)
            System.debug('My job id is: ' + Database.executeBatch(new TestBatchFiringBatch(), 1));

            // Results in a valid CronTrigger id, but schedulable does not execute.
            // CronTrigger Id is not valid after method executes (but it is in test context!)
            System.debug('My schedulable job id is ' + System.schedule('Test Job', '20 30 8 10 2 ?', new TestSchedulableFiring()));

            // This would fail with a `LimitException` - DML is not allowed.
            // Account a = new Account(Name = 'Testy Test Test');
            // insert a;
            // System.debug('My Account\'s Id is ' + a.Id);

            // Setting a savepoint is DML-equivalent - this also throws a `LimitException`
            // System.Savepoint sp = Database.setSavepoint();
        }
    }

Performing DML results in just the `LimitException` expected, as does setting a save point - a DML-equivalent operation. Neither are allowed in this context.

However, the behavior of invoking Schedulable or Batchable classes from a Visualforce controller is stranger. These are both DML-ish operations, involving serializing and persisting the job data. But while neither actually works, we also *don't* get a `LimitException`.

Both `Database.executeBatch()` and `System.schedule()` execute successfully, and each returns an apparently valid `Id` value for an `AsyncApexJob` or `CronTrigger` respectively. These Ids can be output with `System.debug()` and are visible in the logs if you preview the Visualforce page. The corresponding records, however, don't exist following the completion of the controller (not even in a failed state), and the asynchronous jobs do not execute at all. In the batch case, neither `start()` nor `execute()` is called. It's as if the asynchronous job was never invoked, or if the entire constructor is wrapped in a savepoint/rollback structure.

This behavior is unique to the constructor being run as the controller of the Visualforce page. In a test context or Anonymous Apex, with the exact same controller being instantiated directly, both asynchronous jobs complete successfully, and the `AsyncApexJob` and `CronTrigger` records can be inspected afterwards. Both batchable and schedulable jobs can be successfully invoked from an action method, including when the action method is fired by `<apex:page action="{! myAction }">` upon page load.

Now, performing these kinds of operations in a controller constructor is a bad idea (and a security risk) for a wide variety of reasons. It's not a missing capability. Rather, it seems to just be an interesting edge case of undefined behavior on the Salesforce platform, and something to watch out for in Visualforce-land.