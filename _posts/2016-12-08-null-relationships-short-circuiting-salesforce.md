---
layout: post
title: Null Relationships and Short-Circuiting Behavior in Salesforce Formulas, Process Builder, and Apex
---

What happens when you refer to a field across a lookup relationship, and the lookup relationship is `null`? The answer turns out to vary across contexts in some non-obvious ways. In the course of debugging some Process Builder logic, I came up with a summary. In all of the examples below, I'm using a custom object called `Test Base Object` with a nullable lookup relationship `Account__c`.

![Object Setup]({{ site.baseurl }}/public/null-relationship-screens/testobjectsetup.PNG)

## Formula Fields

When a cross-object reference is used in a formula field and the lookup is `null`,
the value of the cross-object reference is also `null`. No exception is thrown.

Boolean logic treats the `null` value as `false`. Since no exception occurs,
the ordering of Boolean clauses is irrelevant and no short-circuit evaluation
is needed to obtain correct results.

## Process Builder

Process Builder's condition and formula-based
triggers *appear* to operate like formulas, but actually handle `null` relationships
very differently. *Dereferencing a `null` field in a Process Builder condition or
formula always results in an exception*. With complex logic in conditions for running actions,
this can be tricky to debug. The errors it produces for users are opaque and
frustrating, preventing any mutation of the involved object.

![Process Builder Error]({{ site.baseurl }}/public/null-relationship-screens/pberror.PNG)

Fortunately, Boolean operators and functions (`AND` and `OR`, including the implicit logical operations used in condition-based triggers) in the Process Builder context perform *short-circuit evaluation*. In other words, references across the lookup relationship can be guarded by checks against `null` lookups earlier in the evaluation order such that evaluation will stop *before* reaching the cross-object relationship, avoiding an exception. The evaluation order is
left-to-right for the `AND()` and `OR()` functions and the `&&` and `||` operators,
and top-to-bottom for condition lists.

![Safe Process Builder Condition Pattern]({{ site.baseurl }}/public/null-relationship-screens/safepbcriteria.PNG)

Using conditions in Process Builder, always precede a cross-object field reference
(assuming a nullable lookup relationship) with a null check. As in this example,
protect the `[Test_Base_Object__c].Account__c.Name` reference with a preceding
"Is null" condition on `[Test_Base_Object__c].Account__c`. Because the criteria
are set to require "All of the conditions are met (AND)", if Condition 1 evaluates to `false` (indicating a `null` value in the lookup field), evaluation of conditions will immediately stop, and no exception will be thrown.

Note that this won't work the same way using an OR condition. OR short-circuits on `true` values,
and short-circuiting because of a `null` lookup is often not the desired behavior.
In many cases, it's easier to handle possible `null` relationships by using customized logic and
nesting an AND with the above null-check within the OR. Constructing a formula
may be more straightforward.

Formulas in Process Builder short-circuit in the same way, whether using the `AND()` and `OR()` functions or the `&&` and `||` operators. The following pattern is safe.

![Safe Process Builder Formula Pattern]({{ site.baseurl }}/public/null-relationship-screens/safepbformula.PNG)

## Apex

In most cases, Apex handles `null` relationships in the same way Process Builder
formulas do. However, there's one variant case.

![Good Apex Code]({{ site.baseurl }}/public/null-relationship-screens/goodapex.PNG)

Accessing the relationship path directly from the queried object simply results in a `null`; no exception is thrown. However, if the intermediate object is assigned to another variable before dereferencing its field, you get a `NullPointerException`.

Like Process Builder formulas, Apex supports short-circuit evaluation. The code below
doesn't throw an exception until line 10, and outputs the representation of `a`, `null`,
and `false`.

![Bad Apex Code]({{ site.baseurl }}/public/null-relationship-screens/badapex.PNG)
