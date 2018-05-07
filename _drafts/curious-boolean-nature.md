---
layout: post
title: The Curious Nature of the Salesforce Boolean
---

The Boolean can be among the simplest data types: it is either true or false, full stop, no complications.

On the Salesforce platform, this couldn't be further than the truth.

Consider the following subtle bug:

    public class BooleanPropertyCheck {
        public Boolean myProperty { get; set; }
        
        // ... more code here ...
    }
    
    @isTest
    public class BooleanPropertyCheckTest {
        @isTest
        public static void testTheThing() {
            BooleanPropertyCheck bpc = new BooleanPropertyCheck();

            System.assert(bpc.myProperty); 
        }
    }
    
That ought to be fine - we're `assert`ing a Boolean value, which is what asserts are for. But here, it's not fine. Instead, we get a `NullPointerException`. This reveals three things, in ascending order:

  1. The value of `myProperty` is `null` when not uninitialized, which is *not* equivalent to `false` in Apex;
  1. The `System.assert()` method throws an exception when passed `null`;
  1. Boolean variables in Apex are actually tri-valued: they can be `true`, `false`, or `null`, and they default to `null`.
  
That's well and good, at a certain level; there's nothing *all that wrong* with tri-valued Booleans, although it's arguable that the behavior of `System.assert(null)` is poorly designed. What's far more confusing is that there are other layers of the Salesforce platform where Booleans are *not* treated as tri-valued.

Consider SOQL. The [SOQL and SOSL Reference](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql_filtering_on_booleans.htm) has this to say on filtering on Booleans:

> You can use the Boolean values TRUE and FALSE in SOQL queries.
> To filter on a Boolean field, use the following syntax:

> `WHERE BooleanField = TRUE`
> `WHERE BooleanField = FALSE`

But we learn [elsewhere](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql_relationships_lookup.htm) in the reference another facet:

> In a WHERE clause that uses a Boolean field, the Boolean field never has a `null` value. Instead, `null` is treated as `false`. Boolean fields on outer-joined objects are treated as `false` when no records match the query.

So Booleans in the database are tri-valued, but in SOQL are treated as binary-valued. This generates rather odd situations like the following,

    List<Custom_Object__c> objects = [SELECT Boolean_Field__c FROM Custom_Object__c WHERE Boolean_Field__c = false];
    
    for (Custom_Object__c o : objects) {
        System.debug(o.Boolean_Field__c);
    }
    
where we'll find a mixture of `false` and `null` values.

The picture grows still more confusing if we look at Hierarchy Custom Settings. Custom Settings, of course, are custom objects, and they can contain checkbox fields. But Hierarchy Custom Settings have a unique feature allowing them to cascade populated field values down the hierarchy (Organization to Profile to User) until overriden at a lower level.

Suppose we have a custom setting `Instance_Settings__c`, with a single Checkbox field `Run__c` and some arbitrary set of other fields.
