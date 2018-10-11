---
layout: post
title: What Can You `GROUP BY`?
---

The Salesforce documentation is notably terse in describing [Considerations When Using `GROUP BY`](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql_select_group_by_considerations.htm). The guidance provided for determining which fields can be grouped is simply:

> The Field object associated with `DescribeSObjectResult` has a groupable field that defines whether you can include the field in a GROUP BY clause.

This is a rather roundabout way to point to the method `DescribeFieldResult.isGroupable()`; for example, `Account.Description.getDescribe().isGroupable()` returns `false`.

Further, the document states that

> You can't use child relationship expressions that use the __r syntax in a query that uses a GROUP BY clause. 

This gives us very little to go on, without examining the Describe information for every single field we might want to group on (doubly challenging in a Dynamic SOQL context). So which field types do, in fact, permit grouping? And does the final sentence prohibit the use of custom relationships in `GROUP BY`?

Failure to use a properly groupable field yields the error

> field 'FIELD_NAME__c' can not be grouped in a query call

It turns out that groupability breaks down pretty cleanly along type lines, with a few interesting nuances. The underlying SOAP type appears to be the primary, but not the sole, determinant. Some fields within the same SOAP type differ in groupability based on other facets. Further, some formula fields *can* be used as groupings - but not the ones you might naively expect from other Salesforce Platform limitations!

Types are listed below by UI type, with the SOAP type in parentheses. This information was derived from inspection of numerous field describes via Workbench and the Tooling API.

## Groupable Field Types

- Checkbox (boolean)
- Phone (string)
- Picklist (string)
- Email (string)
- Text (string)
- Text Area (string)
- URL (string)
- Number (int). Does not include custom fields, only standard Number fields with SOAP type `int`, like `Account.NumberOfEmployees`.
- Lookup (id)
- Id (id)
- Date (date)
- Direct cross-object references to groupable fields, up to 5 levels from the root object (SOQL limit), as in `SELECT count(Id) FROM Contact GROUP BY Account.Parent.Parent.Parent.Parent.Name`. Both custom and standard references are groupable.
- Formulas of type Checkbox and Date, including cross-object formulas across standard and custom relationships.

## Non-Groupable Field Types
 - Address Compound Fields
   - Components of Address compound fields *are* groupable if their types otherwise allow it.
 - Geolocations, both custom and standard, and whether or not defined as having decimal places, including the compound field and components (location/double)
 - Long Text (string)
 - Rich Text (string)
 - Auto Number (string)
 - Multi-Select Picklist (string)
 - Number (double), including custom Number fields with or without decimal and regardless of scale.
 - Percent (double), including custom Percent fields with or without decimal and regardless of scale.
 - Currency (double), including custom Currency fields with or without decimal and regardless of scale.
 - Roll-Up Summary Fields (double), including `COUNT` rollups.
 - Encrypted Text Fields (Classic Encryption; string)
 - Date/Time (dateTime)
 - Time (time)
 - Formulas of types other than Checkbox and Date, **including the otherwise-groupable String type**.

 This post grew out of an interesting question on [Salesforce Stack Exchange](https://salesforce.stackexchange.com/questions/235370/error-thrown-while-trying-to-do-a-group-by/235383#235383). I was intrigued by the lack of definition to this facet of SOQL and spent some time putting together a [community wiki answer](https://salesforce.stackexchange.com/questions/235528/what-types-of-fields-are-groupable-in-a-soql-group-by-clause).