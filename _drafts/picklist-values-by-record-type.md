Unfortunately, picklist + record type information has very uneven coverage across Salesforce APIs.

You can retrieve all picklist values *without* record type information using either Apex or the Tooling API (a single query, in the latter case). 

You can retrieve picklist values *with* record type information, for a single record type at a time, using the UI API, with one callout for each field or one for all fields on the object at once.

You can get all of the information about record types and dependencies by performing a retrieve using the Metadata API that includes the appropriate object and field entries.

# UI API

The UI API lets you make REST requests to source picklist value details.

For example, you can send a GET request to 

> `/services/data/v42.0/ui-api/object-info/Opportunity/picklist-values/<your record type id>`

to get back a JSON bundle covering *all* the picklists and their legal values for that record type, or call

> `/services/data/v42.0/ui-api/object-info/Opportunity/picklist-values/<your record type id>/StageName`

substituting the field API name you're interested in for `StageName`, to get back JSON for just a single field, like this:

    {
      "controllerValues" : { },
      "defaultValue" : null,
      <attributes snipped>,
      "values" : [  
       {
        "attributes" : {
          "closed" : false,
          "defaultProbability" : 0.0,
          "forecastCategoryName" : "Pipeline",
          "picklistAtrributesValueType" : "OpportunityStage",
          "won" : false
        },
        "label" : "New",
        "validFor" : [ ],
        "value" : "New"
      }, {
        "attributes" : {
          "closed" : false,
          "defaultProbability" : 25.0,
          "forecastCategoryName" : "Pipeline",
          "picklistAtrributesValueType" : "OpportunityStage",
          "won" : false
        },

     ... and so on ...

See the UI API documentation for more: 

 - [Get Values for All Picklist Fields of a Record Type
](https://developer.salesforce.com/docs/atlas.en-us.uiapi.meta/uiapi/ui_api_features_records_dependent_picklist.htm)

# Tooling API

Using the Tooling API, you can issue a query like the following:

    SELECT Label, Value FROM PicklistValueInfo WHERE IsActive = true AND EntityParticleId = 'Opportunity.StageName'

In Workbench, this will yield

        Label	                Value
    1   Demo/Pre-proposal       Demo/Pre-proposal
    2   Contract Negotiation    Contract Negotiation
    3   Lead                    Lead
    4   Contract Execution      Contract Execution
    5   Lost                    Lost
    6   On Hold                 On Hold

and so on. Note that Tooling API `EntityParticleId` values are a little nonintuitive. 

# Apex

In Apex, you can call `getPicklistValues()` on any [`DescribeFieldResult`](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_methods_system_fields_describe.htm) for a picklist-type field. Each [`PicklistEntry`](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_class_Schema_PicklistEntry.htm#apex_class_Schema_PicklistEntry) object includes an `isActive()` method you can call. E.g.,

    Schema.DescribeFieldResult dfr = Opportunity.StageName.getDescribe();
    
    for (PicklistEntry pe : dfr.getPicklistValues()) {
        if (pe.isActive()) {
            // do something 
        }
    }

There are some [clever workarounds](https://salesforce.stackexchange.com/questions/4462/get-lists-of-dependent-picklist-options-in-apex?newreg=51a794a0af864ee186a4cc7019a05179) people have developed over the years for getting record type dependency information in Apex, but none of them are supported.

# Metadata API

The Metadata API is a big topic, so this will only be a quick note. When you pull down an object's metadata, you'll get back XML that includes `<recordTypes`> elements, which contain `<picklistValues>` elements:

    <recordTypes>
        <fullName>Test</fullName>
        <active>true</active>
        <businessProcess>Test</businessProcess>
        <description>Test.</description>
        <label>Test</label>
        <picklistValues>
            <picklist>Changes__c</picklist>
            <values>
                <fullName>Approved</fullName>
                <default>false</default>
            </values>
            <values>
                <fullName>Requested</fullName>
                <default>false</default>
            </values>
           ... and so on...

If you also need to ingest the total value set (not just the set for a given record type), you'll need to look further within the object metadata for the relevant `<fields>` entries for the picklist fields themselves:

    <fields>
        <fullName>Changes__c</fullName>
        <externalId>false</externalId>
        <label>Changes</label>
        <picklist>
            <picklistValues>
                <fullName>Approved</fullName>
                <default>false</default>
            </picklistValues>
            <picklistValues>
                <fullName>Requested</fullName>
                <default>false</default>
            </picklistValues>
            ... and so on...

You could write quite a bit of code to parse out and correlate all of that information from the object's XML source (or the return values via your favorite SOAP client/library), the details of which are going to be beyond the scope of just one answer.