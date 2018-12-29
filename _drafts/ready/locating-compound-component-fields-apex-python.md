---
layout: post
title: Locating Salesforce Compound and Component Fields in Apex and Python
---

One of the odder corners of the Salesforce data model is the compound fields. Coming in three main varieties (Name fields, Address fields, and Geolocation fields), these fields are accessible *both* under their own API names and in the forms of their components under their own API names. The compound field itself is always read-only, but the components may be writeable.

For example, on the `Contact` object is a compound address field `OtherAddress`. (There are a total of four standard Address fields spread across the `Contact` and `Account` objects). The components of `OtherAddress` are

 - `OtherStreet` 
 - `OtherCity`
 - `OtherState`
 - `OtherPostalCode`
 - `OtherCountry`
 - `OtherStateCode`
 - `OtherCountryCode`
 - `OtherLatitude`
 - `OtherLongitude`
 - `OtherGeocodeAccuracy`.

So, if we're working in dynamic Apex or building an API client, how do we acquire and understand the relationships between these compound and component fields?

### API

In the REST API, the Describe resource for the sObject returns metadata for the object's fields as well. This makes it easy to acquire all the data we need in one go.

> GET /services/data/v43.0/sobjects/Contact/describe

yields, on a lightly customized Developer Edition, about 250kb of JSON. Included is a list under the key `"fields"`, which contains the data we need (abbreviated here to omit irrelevant data points):

    "fields": [
        {
            "compoundFieldName": null,
            "label": "Contact Id",
            "name": "Id"
        },
        {
            "compoundFieldName": "Name",
            "label": "First Name",
            "name": "FirstName"
        }
    ]

Each field includes its API name (`"name"`), its label, other metadata, and `"compoundFieldName"`. The value of this last key is either `null`, meaning that the field we're looking at is not a component field, or the API name of the parent compound field.

This structure can be processed easily enough in Python or other API client languages to yield compound/component mappings. Given some JSON `response` (parsed with `json.loads()`), we can do

    def get_compound_fields(response):
        return set(
            [field["compoundFieldName"] for field in response["fields"] if field["compoundFieldName"] is not None]
        )

Likewise, we can get the components of any given field:

    def get_component_fields(response, api_name):
        return [field["name"] for field in response["fields"] if field["compoundFieldName"] == api_name]

Both operations can be expressed in various ways, including uses of `map()` and `filter()`, or can be implemented at a higher level if the describe response is processed into a structure, such as a `dict` keyed on field API name.

### Apex

The situation in Apex is rather different because of the way Describe information is returned to us. Rather than a single, large blob of information covering an sObject and all of its fields, we get back individual describes for an sObject (`Schema.DescribeSobjectResult`) and each field (`Schema.DescribeFieldResult`). (We can, of course, call out to the REST Describe API in Apex, but this requires additional work and an API-enabled Session Id).

Moreover, `Schema.DescribeFieldResult` does not include the critical `compoundFieldName` property.

... or rather, it isn't [*documented*](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_methods_system_fields_describe.htm#apex_methods_system_fields_describe) to include it. In point of fact, it does contain the same data returned for a field in the API Describe call, as we can discover by inspecting the JSON result of serializing a `Schema.DescribeFieldResult` record.

Unlike some JSON-enabled magic, we can get to this hidden value in plain Apex. Even though it's undocumented, these references compile and execute as expected:

    Contact.OtherStreet.getDescribe().compoundFieldName

and

    Contact.OtherStreet.getDescribe().getCompoundFieldName()

This makes it possible to construct Apex utilities like we did in Python to source compound fields and compound field components. In Apex, we'll necessarily be a bit more verbose than Python. Performance is a concern in broad-based searches: both finding compound fields on one sObject and locating component fields for one compound field take between 0.07 and 0.1 second in unscientific testing. Your performance may vary.

    public class CompoundFieldUtil {
        public static List<SObjectField> getCompoundFields(SObjectType objectType) {
            Map<String, SObjectField> fieldMap = objectType.getDescribe().fields.getMap();
            List<SObjectField> compoundFields = new List<SObjectField>();
            Set<String> compoundFieldNames = new Set<String>();

            for (String s : fieldMap.keySet()) {
                Schema.DescribeFieldResult dfr = fieldMap.get(s).getDescribe();

                if (dfr.compoundFieldName != null && !compoundFieldNames.contains(dfr.compoundFieldName)) {
                    compoundFields.add(fieldMap.get(dfr.compoundFieldName));
                    compoundFieldNames.add(dfr.compoundFieldName);
                }
            }

            return compoundFields;
        }

        public static List<SObjectField> getComponentFields(SObjectType objectType, SObjectField field) {
            Map<String, SObjectField> fieldMap = objectType.getDescribe().fields.getMap();
            List<SObjectField> components = new List<SObjectField>();
            String thisFieldName = field.getDescribe().getName();
                    
            for (String s : fieldMap.keySet()) {
                if (fieldMap.get(s).getDescribe().compoundFieldName == thisFieldName) {
                    components.add(fieldMap.get(s));
                }
            }
            
            return components;
        }
    }

Then, 

    System.debug(CompoundFieldUtil.getComponentFields(Contact.sObjectType, Contact.OtherAddress));

yields

> 14:15:14:523 USER_DEBUG [1]|DEBUG|(OtherStreet, OtherCity, OtherState, OtherPostalCode, OtherCountry, OtherStateCode, OtherCountryCode, OtherLatitude, OtherLongitude, OtherGeocodeAccuracy)

and 

    System.debug(CompoundFieldUtil.getCompoundFields(Contact.sObjectType));

yields

> 22:15:30:089 USER_DEBUG [1]|DEBUG|(Name, OtherAddress, MailingAddress)

Simple modifications could support the use of API names rather than `SobjectField` tokens, building maps between compound field and components, and similar workflows.

--

This post developed out of a [Salesforce Stack Exchange](https://salesforce.stackexchange.com/questions/244947/get-components-of-a-compound-field/244952#244952) answer I wrote, along with work on a soon-to-be-released data loader project.