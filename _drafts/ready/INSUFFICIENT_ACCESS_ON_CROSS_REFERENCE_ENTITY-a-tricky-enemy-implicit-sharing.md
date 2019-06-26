---
layout: post
title: "`INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY`, A Tricky Enemy: Implicit Sharing, or, when `without sharing` Isn't Enough"
---

In a previous post, we looked at how using `without sharing` can alleviate certain classes of `INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY` exceptions when working with manual sharing in Apex. Today, we'll look at another manual sharing situation where `without sharing` is not enough to solve the issue.

We'll be working in an org that has a Private sharing model for both the Account and the Opportunity objects. We're working on code that performs transfers of Opportunity objects from one user to another - a referrals system, if you will. Trying to head off Apex at the pass, we go ahead and build our service class `without sharing`, and call it from our trigger. The code might look something like this:

```apex
	public without sharing class OpportunityService {
		public static void transferReferralOpportunities(List<Opportunity> newList) {
			List<Opportunity> toUpdate = new List<Opportunity>();

			for (Opportunity o : newList) {
				toUpdate.add(new Opportunity(Id = o.Id, OwnerId = getTransferTarget(o)));
			}

			update toUpdate;
		}
	}
```

But we still get `INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY` when we try to move an Opportunity between one user and another. We're running `without sharing` &mdash; and we're not even inserting an `OpportunityShare`! What's going on here?

The trick here is that the `Opportunity` itself isn't the problem. It's the related `Account`, which is made visible to the owner of the Opportunity via Implicit Parent Sharing.

> [FIXME] Quote on Implicit Parent Sharing.

We can demonstrate this further with a test class, showing exactly which patterns fail and which succeed with synthetic data. Let's go through the scenarios here.

First, let's talk about the data setup we're going to establish in this test class to simulate the problem. We'll create three users (imaginatively named Test 0, Test 1, and Test 2).

```apex
@isTest
public without sharing class Test_Sharing {
	@testSetup
    public static void setup() {
        List<User> users = new List<User>();
        for (Integer i = 0; i < 3; i++) {
            users.add(
                new User(
                    UserName = 'test' + String.valueOf(i) + '@brave-fox-59aa77-dev-ed.my.salesforce.com',
                    LastName = 'Testerson',
                    Alias='brave' + String.valueOf(i),
                    Email='david@ktema.org',
                    ProfileId = [SELECT Id FROM Profile WHERE Name = 'Standard User'].Id,
                    LocaleSidKey = 'en_US',
                    EmailEncodingKey = 'UTF-8',
                    LanguageLocaleKey = 'en_US',
                    TimeZoneSidKey = 'America/Denver'
            	)
            );
        }
        insert users;
        
            
        System.runAs(new User(Id = UserInfo.getUserId())) {
            Account a = new Account(Name = 'Test', OwnerId = users[0].Id);
            insert a;
            Opportunity o = new Opportunity(Name = 'TestCorp', StageName = 'New', CloseDate = Date.today(), AccountId = a.Id, OwnerId = users[1].Id);
            insert o;
        }
    }
```

Once we have our users, we insert some test data: a single Account and a single Opportunity. User Test 0 owns the Account, but Test 1 owns the Opportunity. Since we're operating in a Private sharing model, user Test 2 (who is on the Standard User profile) cannot see either record. User Test 1 has a Parent Implicit Share allowing them to see, but not edit, the Account.

Now, let's look at ways that Opportunity transfers and sharing can fail in this situation.

## Test 1: Share Opportunity to User Test 2

```apex
    @isTest
    public static void shareOpportunityWillFail() {
        User u = [SELECT Id FROM User WHERE UserName LIKE 'test1%'];
        User other = [SELECT Id FROM User WHERE UserName LIKE 'test2%'];
       
        System.runAs(u) {
            Opportunity o = [SELECT Id FROM Opportunity];
            OpportunityShare os = new OpportunityShare(
            	OpportunityId = o.Id,
                UserOrGroupId = other.Id,
                OpportunityAccessLevel = 'Read'
            );
            insert os;
        }
    }
```

It fails. User Test 1 doesn't have permission to provide User Test 2 with the Parent Implicit Share of the Account that they would obtain through gaining visibility on the Opportunity.

## Test 2: Share Opportunity to User Test 2 with Explicit Account Share

```apex
    @isTest
    public static void shareOpportunityWillSucceed() {
        User u = [SELECT Id FROM User WHERE UserName LIKE 'test1%'];
        User other = [SELECT Id FROM User WHERE UserName LIKE 'test2%'];
        
        Account a = [SELECT Id FROM Account];
            
        AccountShare acts = new AccountShare(
            AccountId = a.Id,
            UserOrGroupId = other.Id, // An explicit share to `u` does not affect the outcome
            AccountAccessLevel = 'Read',
            CaseAccessLevel = 'None',
            OpportunityAccessLevel = 'None'
        );
        insert acts;

        System.runAs(u) {
            Opportunity o = [SELECT Id, OwnerId FROM Opportunity];
            
            System.assertEquals(u.Id, o.OwnerId);
            
            OpportunityShare os = new OpportunityShare(
            	OpportunityId = o.Id,
                UserOrGroupId = other.Id,
                OpportunityAccessLevel = 'Read'
            );
            insert os;
        }
    }
```

This version succeeds. Note the sidestep we perform by inserting an Account share to Test User 2 *before* we change user context with `System.runAs()`. At that point, our test is executing as us, a System Administrator, and has the right to perform that share. Once we've done so, User Test 2 can receive the Opportunity share, because they already have visibility on the parent Account.

## Test 3: Transfer Opportunity to User Test 2

```apex
    @isTest
    public static void transferOpportunityWillFail() {
        User u = [SELECT Id FROM User WHERE UserName LIKE 'test1%'];
        User other = [SELECT Id FROM User WHERE UserName LIKE 'test2%'];
        System.runAs(u) {
            Opportunity o = [SELECT Id FROM Opportunity];
            o.OwnerId = other.Id;
            update o;
        }
    }
```

Transferring the Opportunity fails for exactly the same reason as sharing it: our running user still doesn't have the right to expose the Account via a Parent Implicit Share.

## Test 4: Transfer Opportunity to User Test 2 with Explicit Account Share

As with sharing, we can solve the transfer problem by adding an explicit share of the Account to the target user while running as a user privileged to do so.

```apex
    @isTest
    public static void transferOpportunityWillSucceed() {
        User u = [SELECT Id FROM User WHERE UserName LIKE 'test1%'];
        User other = [SELECT Id FROM User WHERE UserName LIKE 'test2%'];
        Account a = [SELECT Id FROM Account];
        
        AccountShare acts = new AccountShare(
            AccountId = a.Id,
            UserOrGroupId = other.Id,
            AccountAccessLevel = 'Read',
            CaseAccessLevel = 'None',
            OpportunityAccessLevel = 'None'
        );
        insert acts;

        System.runAs(u) {
            Opportunity o = [SELECT Id FROM Opportunity];
            o.OwnerId = other.Id;
            update o;
        }
    }
```
# How Do We Fix It?

There isn't a magic bullet to fix this particular problem &mdash; we can't just change the sharing model of our class and have it go away. As we've seeing `without sharing` isn't enough to do the trick.

One solution route is business process-based rather than technological: build your referral or transfer functions such that a higher-level supervisor, such as a Sales leader who's in a position in the role hierarchy to share the parent object that's blocking the transfer, manages the transfer process in lieu of lower-level record owners.

Another is technical. 