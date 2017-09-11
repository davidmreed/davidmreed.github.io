---
layout: post
title: Distinguishing Custom Settings in Apex
---

In many respects, the Salesforce API treats Hierarchy and List Custom Settings the same way: their schemata are the same, and most, though not all, static methods apply to both types of settings. This works well in the typical use case of calling `MyCustomSetting__c.getInstance()`. But suppose you'd like to build generic code that operates on settings. How can you tell list from hierarchy settings to handle them appropriately?

 Custom Settings are easily identifiable with `Schema.DescribeSObjectResult.isCustomSetting()`, but there's no analogue method for custom setting type.  Luckily, there's two critical differences in field behavior on these objects that yield methods of determining which object is which.

 * For a Hierarchy setting, `Name` is nillable, while it is required for List settings.
 * Inserting a List Custom Setting with a non-null `SetupOwnerId` results in a `FieldIntegrityException`.

Hence, a simple way to tell the two apart, given a `DescribeSObjectResult`:

     public Boolean isHierarchyCustomSetting(Schema.DescribeSObjectResult s) {
        return s.isCustomSetting() && s.fields.getMap().get('Name').getDescribe().isNillable();
     }

     public Boolean isListCustomSetting(Schema.DescribeSObjectResult s) {
        return s.isCustomSetting() && !s.fields.getMap().get('Name').getDescribe().isNillable();
     }

An alternate, but slower, route is to construct an instance with `newSobject()` and populate the `SetupOwnerId` field. A (catchable) exception upon insert identifies the class as a List Custom Setting.
