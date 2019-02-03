---
layout: post
title: "`INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY`, A Tricky Enemy: Implicit Sharing, or, when `without sharing` Isn't Enough"
---

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
    
    @isTest
    public static void transferOpportunityWillSucceed() {
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
            Opportunity o = [SELECT Id FROM Opportunity];
            o.OwnerId = other.Id;
            update o;
        }
    }
}```