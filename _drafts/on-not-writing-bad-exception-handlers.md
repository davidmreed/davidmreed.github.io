---
layout: post
title: On Not Writing Bad Exception Handlers
---

Unlike some languages (such as Python) where exceptions have a greater role in flow control, in Apex, exceptions generally are *exceptional*: they connote some relatively uncommon bad state in the application that prevents the normal path of execution from continuing without some special handling. An exception is a message from the application to you, and one that you ignore at your, and your users', peril! 

Because exceptions _are_ exceptional in Apex, they're often frustrating companions early on in the lifecycle of an application, and bugbears that haunt newer developers who haven't yet internalized the patterns that cause them. It's easy to react to those situations by writing defensive Apex: code designed less to behave as expected under the myriad of conditions encountered in production and more to make the compiler and the runtime be quiet. 

This post aims to diagnose several of these patterns (ranging from the unidiomatic and confusing through to those that cause failure and loss of data integrity), draw out their pathologies, and help developers understand _why_ they're not quite right, arming them to write better Apex and better confront failures that do occur.

### Exception Handling over Logic

In Python, this kind of logic is fairly normal:

```python
try:
    process_data(some_dict[key])
except KeyError:
    pass
```

That is, we try something first, catch a (specific) exception if thrown, and either continue execution or perform some other logic to handle the failure.

In Apex, this kind of structure can work, but isn't very idiomatic. Rather than, for example, doing

```apex
try {
    processData(someMap.get(c.AccountId).Parent);
catch (NullPointerException e) {
    // do nothing
}
```

check the state before executing the risky operation - here, accessing a property of a value (the return value of `Map#get()`) that may be `null`:

```apex
if (someMap.containsKey(c.AccountId)) {
    processData(someMap.get(c.AccountId).Parent);
}
```

It's often easier to write unit tests for Apex code that's built this way. And, since Apex does not use exceptions as pervasively for flow control as some languages do, logic of this form is more consistent with the remainder of the application's code.

### Unneeded Exception Handlers

It's easy to get overzealous about handling exceptions in an attempt either to be proactive or simply to get some troublesome code across the finish line. Doing so, however, can actually worsen code. Take these lines, for example:

```apex
List<Account> accountList;

try {
    accountList = [SELECT Id, Name FROM Account];
} catch (Exception e) {
    accountList = new List<Account>();
}
```

At first blush, the handler looks like it could make sense. It's providing a default value for the following logic if the SOQL query should fail, which is what a good exception handler might in fact do.

But the problem here is that the SOQL query shown _cannot throw any catchable exception_. It won't throw a `QueryException` regardless of data volume, because we're assigning to a `List<Account>` rather than an `Account` sObject, and we're not using any of the other SOQL clauses (such as [`FOR UPDATE`](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql_select_for_update.htm) or [`WITH SECURITY_ENFORCED`](https://developer.salesforce.com/docs/atlas.en-us.222.0.apexcode.meta/apexcode/apex_classes_with_security_enforced.htm)) that might cause an exception to be thrown. It can't throw a `NullPointerException`. And if a `LimitException` were thrown, we could not catch it anyway.

Using this pattern worsens our code in several ways:

 1. It increases the complexity of the code while decreasing readability.
 2. It suggests that error handling is taking place, but no error is actually possible.
 3. It reduces unit test code coverage by introducing logic paths that cannot be executed.

(3) is the most pernicious of these effects: the author of this code will never be able to cover the exception handler in a unit test, because the SOQL query cannot be made to throw an exception. As a result, their code coverage will go down, and they'll leave the impression (per (2)) that there are genuine logic paths _not_ being evaluated by their tests.

The solution, of course, is simple: remove the useless exception handler.

## Overbroad Exception Handlers

A similar mistake, but one with a trickier downside, is writing genuine exception handlers that are overbroad in their `catch` blocks. Take the above example with a twist: we add a couple of reasons why the code could throw a real exception.

```apex
List<Account> accountList;

try {
    accountList = [
        SELECT Id, Name, Parent.Name 
        FROM Account 
        WHERE Name = :someAccountName 
        FOR UPDATE
    ];
    accountList[0].Description = 'Child of ' + accountList[0].Parent.Name;
    update accountList;
} catch (Exception e) {
    throw new AuraHandledException('Could not acquire a lock on this account');
}
```

This code can throw two specific exceptions, aside from `LimitException`: `QueryException`, if it tries and fails to obtain a lock on the records that are responsive to the query (for `FOR UPDATE`), and `NullPointerException`, if the first responsive Account does not have `ParentId` populated.

Will this exception handler catch those exceptions? Yes, it will. But it comes with some risks to do so, and it's a better pattern to catch the _specific_ exception you know may be thrown. Catching specific exceptions that you know about prevents your code from silently hiding, or incorrectly handling, extra exceptions you _don't_ know about. In the above example, the exception handler is written to make a user aware of the `QueryException`, but it will silently mask the `NullPointerException`. Other forms of this issue might handle the expected exception but incorrectly handle the other exception thrown, resulting in a difficult-to-debug fault in the application.

There are some contexts where fairly broad exception handlers are desirable. In a controller for a Lightning component (like this example) or Visualforce page, we might wish to catch any species of exception and present a failure message to the user. Even there, however, it's best to keep exception handlers as specific as possible. For example, an `update` DML statement should be wrapped in a `catch (DmlException e)` handler, rather than enclosing a much larger block of code in a fully-generic `catch (Exception e)` block.

Keeping exception handlers focused also makes it easier to define the relevant failure modes and logical paths, and can facilitate construction of unit tests. Since tests that exercise exception handlers are often tricky to build in the first place, it's a net gain to write code that's as testable as possible - even if writing very broad exception handlers can sometimes make it easier to force an exception to be thrown in test context.

## Failing to Roll Back Side Effects

Unhandled exceptions cause Salesforce to rollback the entire transaction. This rollback ensures that inconsistent data are not committed to the database. Handling an exception prevents this rollback from occurring - but code which handles the exception is then responsible for maintaining database integrity.

Here's a pathological example:

```apex
public static void commitRecords(List<Account> accounts, List<Opportunity> opportunities) {
    try {
        insert accounts;
        insert opportunities;
    } catch (DmlException e) {
        ApexPages.addMessage(new ApexPages.Message(ApexPages.Severity.FATAL, 'Unable to save the records');
    }
}
```

This code _attempts_ to do the right thing. It handles a single, specific exception across a narrow scope of operations, and it presents a message to the user (in this case, in a Visualforce context) to indicate what happened.

But there's a subtle issue here. If the `DmlException` is thrown by the _second_ DML operation (`insert opportunities`), the Accounts inserted will _not_ be rolled back. They'll remain committed to the database, even though the user was told the operation failed, and if the user should retry will be inserted again. Depending on the implementation of the Visualforce page, other exceptions could occur due to the stored sObjects being in an unexpected state.

The solution is to use a [savepoint/rollback structure](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/langCon_apex_transaction_control.htm) to maintain database integrity, since we're not allowing Salesforce to rollback the entire transaction:

```apex
public static void commitRecords(List<Account> accounts, List<Opportunity> opportunities) {
    Database.Savepoint sp = Database.setSavepoint();
    try {
        insert accounts;
        insert opportunities;
    } catch (DmlException e) {
        ApexPages.addMessage(new ApexPages.Message(ApexPages.Severity.FATAL, 'Unable to save the records');
        Database.rollback(sp);
    }
}
```

Now, our handler properly maintains database integrity, and does not allow partial results to be committed.

## Swallowing Exceptions

This pattern is far too common, and it's pure poison.

```apex
try {
    // Do some complex code here
} catch (Exception e) {
    System.debug(e.message());
}
```

Exceptions in Apex, again, connote *exceptional circumstances*: something went wrong. The state of the application or the local context is no longer in a cogent state, and can't continue normally. This usually means one of three things, exclusive of `LimitException` (which we can't catch anyway):

 1. The logic is incorrect.
 2. The logic fails to handle a potential data state.
 3. The logic fails to guard against an issue in some external component upon which it relies.
 
When such exceptions occur, your code is no longer in a position to guarantee to the user that their data will be complete, consistent, and correct after it is saved to the database. Suppressing those exceptions by catching them and writing them to the debug log - where no one in production is likely to see them - allows the transaction to complete successfully, even though an unknown failure has occurred and the results can no longer be reasoned about.

That's _really, really bad_. Swallowing exceptions can cause loss of data, corruption of data, desynchronization between Salesforce and integrated external systems, violation of expected invariants in your data, and more and more failures downstream as the application and its users continue to interact with the damaged data set. 

Swallowing exceptions violates user trust and creates subtle, difficult-to-debug problems that often manifest in real-world usage but evade some unit tests - which may show a false positive because exceptions are suppressed! - and simple user acceptance testing. **Don't use this pattern**. 

## A Brief Word on Writing Good Exception Handlers

We've seen four examples of patterns that create poor exception handlers. How can you recognize a good one?

My rubric is pretty straightforward. A good exception handler: 
 
 1. Handles a specific exception that can be thrown by the code in its `try` block.
 2. Handles an exception that cannot be averted by reasonable logic, or which is the documented failure mode of a specific action.
 3. Handles the exception in such a way as to maintain the integrity of the transaction, using rollbacks as necessary.
 
Ultimately, it's key to remember what exception handlers are made to do: they're not to suppress or hide errors, but to provide a cogent logical path to either recovering from an error or protecting the database from its effects. All else follows from rigorous evaluation of this principle.
