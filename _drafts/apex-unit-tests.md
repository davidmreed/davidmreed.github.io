---
layout: post
title: Against 75% Code Coverage 
---

Salesforce prescribes a code coverage metric for every deployment you make to a production system. 

Here's the challenge: prescribing a metric tends to make meeting that metric an end in itself. That's really unfortunately, because code coverage *per se* is a terrible metric for testing efficacy. Testing has a purpose beyond satisfying the requirements of the Salesforce platform: your tests should be providing a qualitative statement about the behavior of your code, and they should offer a meaningful defense against regressions by showing that these statements continue to be true as your codebase evolves.

Quantitative metrics are seductive. It's easy to focus on the metric to exclusion of the metric's purpose. But you can't get to good tests by counting lines of code covered, or even counting assertions made. Test quality is ultimately both qualitative and highly specific to a codebase. I tend to be less religious about the exact structure of my tests - I'm not a stickler, for example, about using test data factories, and while I use mocking and dependency injection when I need to, I don't find pervasive mocking frameworks especially interesting - and much more religious about the qualitative statements that my tests are making.

I can't even count how many unit tests I've read on Salesforce that do nothing more than call every method in a class to get code coverage. That's the downside of forcing the quantitative testing metric: it becomes a chore; one does what one must to hit the metric and move on, ignoring the ultimate purpose behind the metric. 

Writing good unit tests is a big, complex topic. I'm not going to talk here about the *mechanisms* of writing good unit tests. There's lots of techniques and options at different scales of infrastructure investment, from the simplest assertion-based tests to complex and heavily-abstracted behavior driven development with pervasive mocking and dependency injection. Different organizations and codebases belong at different points on that spectrum - the decision has to do with developer resources, risk tolerance, and the scale and complexity of the codebase. What I want to focus on is this question: *what are the qualitative claims being made by a good unit test?*

When I look at unit tests, I'm asking a series of questions about what's being done.

  - what are the code paths? 



Does the test exercise...

  - a bulk code scenario? 
 
 Typically, for a Salesforce unit test, this means running through each code path (and every trigger event, if applicable) on an input record set of 10+ records. More is definitely better to get a sense of real-world behavior and performance, but a true volume test on, for example, a 200-record trigger context or a 200+-record batch Apex invocation is generally not possible due to limitations on the amount of data you can create in an `@testSetup` method.

There are cases where a bulk test isn't meaningful. Most Visualforce action methods, for example, don't have an applicable bulk test.

  - a single-record scenario?

  A single-record test can seem superfluous if bulk is already being covered, and in some cases it may well be. I like to cover it, though, because there are cases where receiving exactly one record can produce aberrant behavior.

 - a null or empty-list scenario, or, preferably, both? 
 
  It's always critical to determine the behavior of code that's given a null or empty input parameter (or both, for collection parameters). Issues with null handling tend to manifest fairly obviously in `NullPointerException`s. Making sure to test with empty parameters (empty lists and maps, `''` strings) as well helps catch subtler logic errors that don't fail as loudly. Both are necessary.

 - positive and negative functionality?

  It's important to establish that code does what it purports to do. In a shared database environment, though, it's just as important to demonstrate that it doesn't do what it does not purport to do and what it purports not to do (two different qualities!) Take a method like this, for example:

    public static void queryAndUpdateOpportunitiesWithOwners(List<Id> accountIds) {
    	List<Opportunity> opps = [SELECT Id FROM Opportunity];

    	// ... update records ...

    	update opps;
    }
  
  A good test will exercise this functionality with a single-record list, a bulk list, an empty list, and a null list. And most likely, all four will pass, even though the SOQL query in this method is incorrect. That's why it's critical to test behavior with input records that should be unaffected. If, like this method, the code performs SOQL queries or DML, it's also critical to test it in a situation where the database contains non-matching records that shouldn't be affected, and to verify that those records that shouldn't be affected *weren't* affected with appropriate assertions. 

  It's very easy for logic errors and underselective SOQL queries to slip by positive test cases. It's natural for `@testSetup` methods to insert carefully tailored data sets that match the conditions you're looking for in your tests. But if your SOQL isn't correctly limited or your matching logic is faulty, you could potentially push a data corruption bug - or worse - to production. A negative case will help you ferret out those issues early.

  ## 

  Virtuous decisions throughout a codebase and a development lifecycle are mutually reinforcing. Writing good, well-structured tests encourages writing good, well-structured code, because it's harder and less elegant to test poorly-structured code. Similarly, good, well-structured code tends to suggest its own tests, and makes implementing them very quick. 

  Insisting on qualitative guarantees about code behavior forces good tests, but it also forces good habits. 