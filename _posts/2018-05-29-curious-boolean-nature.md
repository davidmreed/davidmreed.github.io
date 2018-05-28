---
layout: post
title: The Curious Nature of the Salesforce Boolean
---

The Boolean can be among the simplest data types: it is either true or false, full stop, no complications.

On the Salesforce platform, this could hardly be further than the truth.

Consider the following bug, in a simplified example:

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

That ought to be fine - we're `assert`ing a Boolean value, which is what asserts are for. (Although one might reasonably expect the assertion itself to fail here). But in fact we get not an assertion failure or success, but a `NullPointerException`. This reveals several things, both specific and broad:

  1. The value of `myProperty` is `null` when not initialized;
  1. `null` is *not* equivalent to `false` in Apex;
  1. The `System.assert()` method throws an exception when passed `null`, but *not* a failed assertion as such;
  1. Boolean variables in Apex are actually tri-valued: they can be `true`, `false`, or `null`, and they default to `null`.

Note that it doesn't matter here whether we define `myProperty` using property syntax or as a simple Boolean instance variable. The behavior is the same.

It's the case in other contexts as well that referencing a `null` Boolean value produces a `NullPointerException` rather than evaluating to `false`. (This can be particularly confusing to debug, as we're conditioned to look for *property access* when troubleshooting a `NullPointerException`). These conditionals, for example, produce `NullPointerException`:

    Boolean b = null;

    System.debug(b ? 'true' : 'false'); // NullPointerException
    if (b) System.debug('b is true!'); // NullPointerException

At a certain level, this is fair enough. We have some weird behaviors to look out for, but there's nothing *all that wrong* with tri-valued Booleans (although it's arguable that the behavior of `System.assert(null)` in particular is poorly designed). We should never rely on uninitialized values in our classes or local variables anyway, but always explicitly initialize them. 

What's perhaps more confusing, though, is that there are other layers of the Salesforce platform where Booleans are *not* treated as tri-valued.

## Booleans and SOQL

Consider SOQL. The [SOQL and SOSL Reference](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql_filtering_on_booleans.htm) has this to say on filtering on Booleans:

> You can use the Boolean values TRUE and FALSE in SOQL queries.
> To filter on a Boolean field, use the following syntax:
> `WHERE BooleanField = TRUE`
> `WHERE BooleanField = FALSE`

But we learn [elsewhere](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql_relationships_lookup.htm) in the reference another facet:

> In a WHERE clause that uses a Boolean field, the Boolean field never has a `null` value. Instead, `null` is treated as `false`. Boolean fields on outer-joined objects are treated as `false` when no records match the query.

So Booleans in the database and in Apex are tri-valued, but in SOQL are treated as binary-valued. A filter in SOQL against `null` is treated exactly like one written against `false`, so the following queries are treated as equivalent:

    [SELECT Id FROM Boolean_Object__c WHERE Boolean__c = false]
    [SELECT Id FROM Boolean_Object__c WHERE Boolean__c = null]

Given this sort of almost-mismatch between SOQL and Apex, one might suppose that one could see the following happen, if we really drive a wedge into the gap:

    Boolean_Object__c b = new Boolean_Object__c();

    System.assertEquals(null, b.Boolean__c); // Should pass
    b.Boolean__c = null;
    System.assertEquals(null, b.Boolean__c); // Should pass

    insert b;

    Boolean_Object__c c = [SELECT Boolean__c 
                           FROM Boolean_Object__c
                           WHERE Id = :b.Id];

    System.assertEquals(b.Boolean__c, c.Boolean__c); // Should fail.

But in fact we don't, because sObject instances don't behave like other Apex classes. In fact, assertion 1 fails, because Booleans are initialized not to `null` but to `false` in sObject class instances. Further, we'll get a `DMLException` at `insert b`, because `null` is not treated as a legal value for inserting Boolean (Checkbox) fields. We can't actually create a situation where there's a `null` Boolean in the database.

So Booleans are tri-valued in Apex, *but are clamped to `true`/`false` around DML statements and SOQL queries*, and there are some special-case behaviors to remain aware of. In particular, users of wrapper or shadow Apex classes from which sObject instances are ultimately generated should keep in mind that the Boolean initialization behaviors differ between the two class types.

## Consequences: Hierarchy Custom Settings

One area where this curious Boolean behavior has practical consequences is the Hierarchy Custom Setting. Custom Settings, of course, are custom objects, and they can contain checkbox fields. But Hierarchy Custom Settings have a unique feature allowing them to cascade populated field values down the hierarchy (Organization to Profile to User) until overriden by a *non-null value* at a lower level.

Suppose we have a custom setting `Instance_Settings__c`, with a single Checkbox field `Run__c` and some arbitrary set of other fields - say `Test__c`, a text field.

If we populate these fields at the Organization level (`SetupOwnerId` = `UserInfo.getOrganizationId()`), we expect rightly that the values at the Organization level will cascade down to the User level, if they're not overriden. And for `Test__c`, our text field, that's true. If we set that field to `"foo"` at the Organization level, and `null` at the User level, sure enough our `Instance_Settings__c.getInstance()` value will inherit the Organization's value `"foo"`. 

But Booleans work differently, because they're treated as binary-valued here - they're never `null`. So a `true` value for `Run__c` will never cascade down to the User or Profile level of our Custom Setting. The instances at those levels always have `false` set for that Checkbox field if we don't explicitly populate `true` upon creation *at that setup level*.

    Instance_Setting__c s = new Instance_Setting__c();
    s.Test__c = 'So say some';
    s.Run__c = true;
    s.SetupOwnerId = UserInfo.getOrganizationId();
    insert s;

    Instance_Setting__c instance = Instance_Setting__c.getInstance();

    System.assertEquals('So say some', instance.Test__c); // Passes
    System.assert(instance.Run__c); // Passes

    s = new Instance_Setting__c();

    s.SetupOwnerId = UserInfo.getUserId();
    s.Test__c = 'So say we all';
    insert s;

    instance = Instance_Setting__c.getInstance();

    System.assertEquals('So say we all', instance.Test__c); // Passes
    System.assert(instance.Run__c); // Fails

## The Upshot is...

 - Booleans are more complicated that one might expect. 
 - Never rely on behavior around uninitialized variables.
 - `NullPointerException` can arise even without a dot-notation object dereference.