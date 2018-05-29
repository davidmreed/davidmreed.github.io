---
layout: post
title: Dependency Injection with Batch Apex
---

Three techniques for dependency-injection testing in a batch class

1. Execute the batch without calling `Database.executeBatch()` - call `execute()` and friends directly.
2. Supply a mock that already expects specific calls, and ensure that a finalize method is called - either at the end of `execute()` for a non-Stateful batch or at the end of `finish()` for a Stateful batch.
3. Use some form of serialization to pass the completed Mock back to the test class, which could access it after `Test.stopTest()`.