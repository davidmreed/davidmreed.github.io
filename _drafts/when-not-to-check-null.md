---
layout: post
title: Defensive Error Checking Gone Wrong
---

Starting out as a Salesforce developer is difficult. 

The Apex language (and far more so, its runtime environment) isn't as simple as it seems, and it doesn't come with some of the niceties that modern general-purpose programming languages have. The mind-share penetration of recent _massive_ improvements in developer tooling still isn't as high as it could be - and for junior developers, those tools are just another of a thousand things to learn!

So too often, junior developers finds themselves confronting what feels like a barrage of errors whose causes they don't intuit, without the tools or conceptual frameworks or available work time to adequately dissect them. And they react by writing defensive Apex: code designed less to behave as expected under the myriad of conditions encountered in production and more to make the compiler and the runtime be quiet.

Sadly, that understandable defensiveness leads to some patterns that range from useless to deceptive to outright harmful. This post aims to diagnose several of these patterns, draw out their pathologies, and help developers understand _why_ they're not quite right, arming them to write better Apex and better confront failures that do occur.

## Checking for Impossible Nulls and Nonexistent Errors

Here's a common example of error checking that doesn't need to be there when working with DML operations:

```apex
List<Account> accounts = new List<Account>();

for (Account a : [SELECT Id, ... FROM Account WHERE ...]) {
    // Do something with `a`, populate `accounts`.
}

if (accounts != null && accounts.size() > 0) {
    update accounts;
}
```
    
This check - `if (accounts != null && accounts.size() > 0)` - is not particularly harmful, but it is useless and decreases the readability of the code. There's no possibility that `accounts` is `null` in this code, because it has been initialized to a value that is guaranteed not to be `null`. Apex isn't C; allocating an object will never yield a `null` value.

Further, there's no need to check whether there are any Accounts in the list before performing DML. DML on an empty collection is a no-op: nothing happens, and no limits are consumed.

So these three lines, and every three lines resembling them, can be replaced with a simple

```apex
update accounts;
```

This change clarifies the logic, reduces the cyclomatic complexity of the code, and removes the *impression* of error checking being performed when in fact no error is possible.

## Overzealous Exception Handling and Hiding

Unlike some languages (such as Python) where exceptions have a greater role in flow control, in Apex, exceptions generally are *exceptional*: they connote some relatively uncommon bad state in the application that prevents the normal path of execution from continuing without some special handling. An exception is a message from the application to you, and one that you ignore at your, and your users', peril! 

There's a few ways that exception handling is misused in Apex. Some are merely unidiomatic or confusing; others create unneeded complexity; and a few actually cause failure and loss of data integrity. Here's three examples. 

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

It's generally easier to write unit tests for Apex code that's built this way. 

Virtually all `NullPointerException` scenarios should be handled via logic, not exception handlers. 

### Unneeded or Overbroad Exception Handling

Two related issues stem from misunderstanding of _when_ and _why_ exceptions are thrown in Apex. Take this code, for example:

```apex
List<Account> accountList;

try {
    accountList = [SELECT Id, Name FROM Account];
} catch (Exception e) {
    accountList = new List<Account>();
}
```

The SOQL query shown here _cannot throw any catchable exception_. It won't throw a `QueryException` regardless of data volume because we're assigning to a `List<Account>` rather than an `Account` sObject, and we're not using any of the other SOQL clauses that might cause an exception to be thrown. It can't throw a `NullPointerException`. And if a `LimitException` were thrown, we could not catch it anyway.

Using this pattern worsens our code in several ways:

1. It increases the complexity of the code while decreasing readability.
2. It suggests that error handling is taking place, but no error is actually possible.
3. It reduces unit test code coverage by introducing logic paths that cannot be executed.

(3) is the most pernicious of these effects: the author of this code will never be able to cover the exception handler in a unit test, because the SOQL query cannot be made to throw an exception. As a result, their code coverage will go down, and they'll leave the impression (per (2)) that there are genuine logic paths _not_ being evaluated by their tests.

The solution, of course, is simple: remove the useless exception handler.

Another species of the same mistake is writing genuine exception handlers that are overbroad in their `catch` blocks. Take the above example with a twist: we add a couple of reasons why the code could throw a real exception.

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
    // ... do something ...
}
```

This code can throw two specific exceptions, aside from `LimitException`: `QueryException`, if it tries and fails to obtain a lock on the records that are responsive to the query (for `FOR UPDATE`), and `NullPointerException`, if the first responsive Account does not have `ParentId` populated.

Will this exception handler catch those exceptions? Yes, it will. But it comes with some risks to do so, and it's a better pattern to catch the _specific_ exception you know may be thrown. Catching specific exceptions that you know about prevents your code from silently hiding, or incorrectly handling, extra exceptions you _don't_ know about. In the above example, if the exception handler is written to manage the `QueryException`, it may either silently mask the `NullPointerException`, or may handle it incorrectly, resulting in a difficult-to-debug fault in the application.

There are some contexts where fairly broad exception handlers are desirable. For example, in a controller for a Lightning component or Visualforce page, we might wish to catch any species of exception and present a failure message to the user. Even there, however, it's best to keep exception handlers as specific as possible. For example, an `update` DML statement should be wrapped in a `catch (DmlException e)` handler, rather than enclosing a much larger block of code in a fully-generic `catch (Exception e)` block.

Keeping exception handlers focused also makes it easier to define the relevant failure modes and logical paths, and can facilitate construction of unit tests. Since tests that exercise exception handlers are often tricky to build in the first place, it's a net gain to write code that's as testable as possible.

### Swallowing Exceptions

This pattern is far too common, and it's pure poison.

```apex
try {
    // Do some complex code here
} catch (Exception e) {
    System.debug(e.message());
}
```

Exceptions in Apex, again, connote *exceptional circumstances*: something went wrong. The state of the application or the local context is typically no longer in a cogent state, and can't continue normally. This usually means one of three things, exclusive of `LimitException` (which we can't catch anyway):

 1. The logic is incorrect.
 2. The logic fails to handle a potential data state.
 3. The logic fails to guard against an issue in some external component upon which it relies.
 
When such exceptions occur, your code is no longer in a position to guarantee to the user that their data will be complete, consistent, and correct after it is saved to the database. Suppressing those exceptions by catching them and writing them to the debug log - where no one in production is likely to see them - allows the transaction to complete successfully, even though an unknown failure has occurred. 

That's _really, really bad_. Swallowing exceptions can cause loss of data, corruption of data, desynchronization between Salesforce and integrated external systems, violation of expected invariants in your data, and more and more failures downstream as the application and its users continue to interact with the damaged data set. 

Swallowing exceptions violates user trust and creates subtle, difficult-to-debug problems that often manifest in real-world usage but evade unit tests - which may show a false positive because exceptions are suppressed! - and simple user acceptance testing. **Don't use this pattern**. 

## Converse: Writing Good Exception Handlers
 
My rubric for when you should write an exception handler is pretty simple: you should write an exception handler if and only if you know that a specific piece of code 
 
 1. Can throw a _specific_ exception.
 2. Cannot avert the exception by reasonable logic, or the exception is the documented failure mode of a specific action.
 3. Can be meaningfully handled while maintaining the integrity of the transaction.

