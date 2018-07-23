---
layout: post
title: Testing the Connect API without `seeAllData=true` 
---

One of the key limitations of the Connect API, which provides interaction with Chatter inside Apex, is that unit tests of Connect API code typically [require the use of `seeAllData=true`](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/connectAPI_TestingApex.htm):

> Chatter in Apex methods don’t run in system mode, they run in the context of the current user (also called the context user). The methods have access to whatever the context user has access to. Chatter in Apex doesn’t support the `runAs` system method.

> Most Chatter in Apex methods require access to real organization data, and fail unless used in test methods marked `@IsTest(SeeAllData=true)`.

While a few Connect API methods provide dedicated `setTest...()`-style methods to supply standardized data, others simply cannot be called at all in test context without the `seeAllData=true` annotation - something all good Apex programmers avoid like the plague!

In some very simple cases, it may be enough to guard a Connect API call with `if (!Test.isRunningTest()) { }`. If used as a one-liner, it'll even still receive code coverage. But, of course, this abrogates our duty to validate code behavior in our unit tests. To do that, we need to apply dependency injection so that we can see what our code under test is doing, without actually invoking the Connect API.

Here's a simple and very explicit example. Note that I'm not using a mocking library or `StubProvider` here. This is a simple example, but I often prefer in real code that's not too complex to use this explicit pattern as well.

The outer class `PostsToChatter` represents our class that calls the Connect API.

We establish an inner interface, `ChatterPoster`, that must be implemented by our mock or delegate class, an instance of another class we'll pass in that does all the action work of calling the Connect API. Think of it as a very thin abstraction layer - at times, using this pattern can even help genericize code to, for example, support notifications over either Chatter *or* email, simply by supplying a different delegate.


    public class PostsToChatter {
        public interface ChatterPoster {
            void postToChatter(Id parentRecord, String text);
        }
     
        @TestVisible private class RealChatterPoster implements ChatterPoster {
            public void postToChatter(Id parentRecord, String text) {
                if (!Test.isRunningTest()) callConnectAPIHere(parentRecord, text);
                // Note that if we keep this minimal (one line) we still obtain full code coverage even though the Connect API call does not actually go through.
            }
        }

        @TestVisible private ChatterPoster myPoster;

        // ... constructors and code ...

        public PostsToChatter() {
            // Add a default delegate (the real Chatter poster class
            myPoster = new RealChatterPoster();
        }

        // In real code, we'd likely have a constructor accepting a `ChatterPoster` parameter too.

        public void doSomethingRequiringAChatterPost() {
            // ... stuff happens ...
            myPoster.postToChatter(someId, someText);
        }
    }

Then, in test context, we have the freedom to inject a completely different class that implements `PostsToChatter.ChatterPoster`, but doesn't really call the Connect API. Meanwhile, because we've kept our actual-Chatter delegate so tiny and simple (and because we've included `Test.isRunningTest()` guards), *we can still call it in test context* to get the code coverage, even though we can't validate its behavior like we ought to.

Here's an example of what one approach to this test class could look like. (This omits the boilerplate test case to cover the `RealChatterPoster` inner class). In this simple implementation, our mock poster makes assertions when called about its own parameters. A more advanced implementation would store them for validation by the test class.

    @isTest
    public static class PostsToChatterTEST {
        private class MockChatterPoster implements PostsToChatter.ChatterPoster {
            public Id expectedId;
            public String expectedString;
            public Integer timesCalled = 0;

            public void postToChatter(Id parentRecord, String text) {
                System.assertEquals(expectedId, parentRecord, 'correct parent');
                System.assertEquals(expectedString, text, 'correct post body');
                timesCalled++;
            }
        }
        
        @isTest
        public static void testTheClass() {
            PostsToChatter p = new PostsToChatter();
            p.myPoster = new MockChatterPoster();
            
            // ... invoke class functionality...
             
            System.assertEquals(1, p.myPoster.timesCalled, 'chatter post called');
            // Post contents are validated in the mock - or store the supplied parameters 
            // in the mock and validate them here, if preferred.
        }
    }

Avoiding `seeAllData=true` is still possible! It's true that many real-world Chatter applications will involve more complex uses of the Connect API, and it will take work to apply the dependency-injection approach. The refactoring effort is likely to pay off, however, in quicker, easier, more reliable testing, and easier extensibility for the future.

And for many straightforward uses of Chatter, like posting of notifications, code much like the above will be enough to cover the testing needs.