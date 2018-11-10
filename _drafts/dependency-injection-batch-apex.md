---
layout: post
title: Dependency Injection with Batch Apex
---

Dependency injection is a powerful tool for unit testing complex, data-dependent code. It's an example of an inversion of control pattern, where an external dependency or reference of the code under test is supplied to that code under test in the form of a delegate object. In test context, the real delegate is replaced with a mock of some species or other, allowing the unit test both deep control of the tested code's operation and visibility into its operation.

Performing dependency injection on Batch Apex classes comes with some unique challenges.

## Direct Synchronous Execution

With direct synchronous execution, we use a standard mock, but avoid transaction-boundary issues by execute the batch directly. Instead of issuing `Database.executeBatch()` inside `Test.startTest()` and `Test.stopTest()`, we call `start()`, `execute()`, and `finish()` directly.

## Mock with Expectations and Finalization Method

Supply a mock that already expects specific calls, and ensure that a finalize method is called - either at the end of `execute()` for a non-Stateful batch or at the end of `finish()` for a Stateful batch.

## Passing Serialized Data Back to Test Class

Use some form of serialization to pass the completed Mock back to the test class, which could access it after `Test.stopTest()`.