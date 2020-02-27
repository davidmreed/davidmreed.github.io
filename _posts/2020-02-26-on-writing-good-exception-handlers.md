---
layout: post
title: On Writing Good Exception Handlers
---

> This post is adapted from an [answer](https://salesforce.stackexchange.com/a/295713/46017) I wrote on [Salesforce Stack Exchange](https://salesforce.stackexchange.com/), and follows a previous [discussion](https://www.ktema.org/2020/01/05/on-not-writing-bad-exception-handlers/) about what makes a *bad* exception handler.

The `try`/`catch` construct might be a little bit misnamed: catching exceptions doesn't matter very much. What matters is what you choose to do with the exception once you've caught it. I might like the construct better if it were called `try`/`handle`.

What it means to handle an exception is to take an *exception*al situation - something bad and out of the ordinary happened - and allow the application to safely move back into an anticipated pathway of operation, preserving

- The integrity of the data involved.
- The experience of the user.
- The outcome of the process, if possible.

Exceptional situations, as befits them, are often very case-specific, which can make it challenging to talk about principles that apply broadly. I like to approach it with two lenses in mind: one is understanding where we're operating in the application relative to the user and the user's understanding of what operations are taking place, and the other is situating the error relative to the commitments we must make to data integrity and user trust.

Let's look at a couple of examples of how this plays out in different layers of a Salesforce implementation.

## Triggers

Suppose we're writing a trigger. The trigger takes data modified by the user, processes it, and makes updates elsewhere. It performs DML and does not use partial-success methods, such as `Database.update(records, false)`. This means that failures will throw a `DmlException`. (If we did use partial-success methods, the same principles apply, they just play out differently because errors return to us in Result objects instead of exceptions).

Here, we have to answer at least two critical questions:

- Are the failures we encounter amenable to being fixed?
- What is the nature of the data manipulation we're doing? If it fails, does that mean that the entire operation (including the change the user made) is invalid? That is, if we allow the user changes to go through without the work we're doing, have we harmed the integrity of the user data?

These questions determine how we'll respond to the exception.

If we know a particular exception can be thrown in a way that we can fix, our handler should just fix it and try again. That would be a genuine "handling" of the exception. In Apex, where exceptions aren't typically used as flow control, this situation is somewhat less common than in a language like Python, but it does occur. For example, if we're building a Queueable Apex class that's designed to gain exclusive access to a particular record using a `FOR UPDATE` clause, we might catch a `QueryException` (indicating that we weren't able to obtain the lock) and handle that exception by chaining into another Queueable, allowing us to successfully complete processing once the record becomes available.

But in most cases, that's not our situation when building in Apex. It's the second question, about data integrity, that tends to be determinative of the appropriate implementation pattern, and it's why I advocate for eschewing exception handlers in many cases.

The most important job our code has is not to damage the integrity of the user's data. For that reason, in most cases where an exception is related to data manipulation, I advocate for not catching it in backend code at all unless it can be meaningfully handled. Otherwise, it's best to let higher-level functionality (below) catch it, or allow the exception to remain unhandled and to cause the whole transaction to be rolled back, preserving data integrity.

To make this concrete: suppose we're building a trigger whose job is to update the dollar value of an Opportunity when the user updates a related Payment. Our Opportunity update might throw a `DmlException`; what do we do? 

Ask the questions: Can we fix the problem in Apex alone? No, we can't. 

If we let the Opportunity update fail while the Payment update succeeds, do we lose data integrity? Yes. The data will be wrong, and invariants that our business users rely upon will be violated.

Let the exception be raised and dealt with at a higher level, or allowed to cause a rollback.

> ἐὰν μὴ ἔλπηται ἀνέλπιστον, οὐκ ἐξευρήσει, ἀνεξερεύνητον ἐὸν καὶ ἄπορον<br />
> Should one not expect the unexpected, one shan't find it, as it is hard-sought and trackless.<br />
> — Heraclitus, Fragment 18

### Non-Critical Backend Functionality

There are other cases where we'll want to catch, log, and suppress an exception. Take for example code that sends out emails in response to data changes (I'll save for another time why I think that's a terrible pattern). Again, we'll look to the questions above:

- Can we fix the problem? No.
- Does the problem impact data integrity if we let it go forward? Also no.

So here is a situation where it might make sense to wrap the sending code in a `try`/`catch` block, and record email-related exceptions using a logging framework so that system administrators can review and act upon them. Then, we don't re-raise the exception - we consume it and allow the transaction to continue.

We probably don't want to block a Case update because some User in the system has a bad email address!

## User-Facing Functionality

Now, turn the page to the frontend - Visualforce and Lightning. Here, we're building in the controller layer, mediating between user input and the database.

We present a button that allows the user to perform some complex operation that can throw multiple species of exceptions. What's our handling strategy?

Here, it is much more common, and even preferable, to use broader `catch` clauses that don't fix the error, but do handle the exception in the sense of returning the application to a safe operating path. They do this by performing an explicit rollback to preserve data integrity and then surfacing a friendly error message to the user - an error message that helps to contextualize what may be a quite low-level exception. 

For example, in Visualforce, you might do something like this:

```apex
Database.Savepoint sp = Database.setSavepoint();
try {
    doSomeVeryComplexOperation(myInputData);
} catch (Exception e) { // Never otherwise catch the generic `Exception`!
    Database.rollback(sp); // Preserve integrity of the database.
    ApexPages.addMessage(new ApexPages.Message(ApexPages.Severity.FATAL, 'An exception happened while trying to execute the operation.'));
}
```

That's friendly to the user, applying that lens of understanding how _they_ scope an operation that may fail or succeed: it shows them that the higher-level, semantic operation they attempted failed (and we might want to include the actual failure message too to give them a shot at fixing it, if we do not otherwise log the failure for the system administrators). But it's also friendly to the database, because we make sure our handling of the exception doesn't impact data integrity. 

Even better would be to be specific about the failure using multiple `catch` blocks (where applicable):

```apex
Database.Savepoint sp = Database.setSavepoint();
try {
    doSomeVeryComplexOperation(myInputData);
} catch (DmlException e) { 
    Database.rollback(sp);
    ApexPages.addMessage(new ApexPages.Message(ApexPages.Severity.FATAL, 'Unable to save the data. The following error occurred: ' + e.getMessage()));
} catch (CalloutException e) {
    Database.rollback(sp); 
    ApexPages.addMessage(new ApexPages.Message(ApexPages.Severity.FATAL, 'We could not reach the remote system. Please try again in an hour.');
}
```

In Lightning (Aura) controllers, we'd re-throw an `AuraHandledException` instead of using `ApexPages.addMessage()`, but the same structural principles apply to how we think about these handlers.

## The One Thing Not To Do

This:

```
try {
    // do stuff
} catch (Exception e) {
    System.debug(e);
}
```

I've written [previously](https://www.ktema.org/2020/01/05/on-not-writing-bad-exception-handlers/) about what a dangerous and bad practice this is. To put it in the context of this discussion, let's consider how we'd have to answer the questions above to make this pattern a good response. 

You'd want to swallow an exception like this when:

- We cannot fix the error.
- What we're doing has no impact on data integrity.

*and* 

- No one is going to notice or care that the functionality isn't working (since we're not telling anyone).

If these three statements are all true, I suspect a design fault in the system!
