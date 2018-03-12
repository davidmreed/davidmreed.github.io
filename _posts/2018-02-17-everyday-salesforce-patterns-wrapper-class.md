---
layout: post
title: "Everyday Salesforce Patterns: The Wrapper Class"
---

The Salesforce platform has great reference documentation, great intro training through Trailhead, and some excellent books and resources on enterprise design patterns. What's less-canonically covered, in the resources I'm familiar with, are everyday patterns: the idiomatic implementation tools that are used and adapted every day by experienced developers. I want to make a contribution to filling this void with this series, starting with some discussion of wrapper classes.

It's extremely common in Visualforce development to need to perform some transformation on data queried out of Salesforce before displaying that data. This can include things like enriching one object with data from another (which is not its parent or child), 'framing' or presenting multiple unrelated objects in a single flat list, such as an `<apex:pageBlockTable>`, applying a mapping table to values in an object's fields, or appending summary data calculated in Apex.

An apt solution to all of these needs is the *wrapper class* pattern. Every wrapper class looks a little different, because it's highly specific to the individual use case and Visualforce page. The overall pattern often looks rather like this example, which wraps either a Contact or a Lead in an inner class, called `Wrapper`, of the page controller. Wrapper classes do not have to be inner classes, but the use of an inner class is common and effective.

Wrapper classes are in some ways similar to *structures*, *union types*, or *algebraic types* provided by other languages, to the extent such patterns are possible with the limited introspection and dynamism available in Apex.

    public with sharing class ApexController {
        public class Wrapper implements Comparable {
            public Lead ld { get; private set; }
            public Contact ct { get; private set; }
            // Included for easier conditional rendering.
            public String dataType { get; private set; }
            public String calculatedTitle { get; private set; } 
            // and so on... include more calculated fields here.

            public Wrapper(Lead l) {
                ld = l;
                dataType = 'Lead';
                // Note that we are assuming the Name field is queried.
                calculatedTitle = 'Lead: ' + ld.Name; 
            }

            public Wrapper(Contact c) {
                ct = c;
                dataType = 'Contact';
                calculatedTitle = 'Contact: ' + ct.Name;
            }

            public Integer compareTo(Object other) {
                Wrapper o = (Wrapper)other;

                // Perform some comparison logic here, such as comparing 
                // the `SystemModStamp` of the embedded sObjects, or 
                // comparing the `calculatedTitle` properties.
                // You can even sort by type by inspecting dataType field, 
                // or, if storing sObject instances, with their 
                // `sobjectType` field.

                return 0;
            }
        }

        // Declare our public property for Visualforce as a list of 
        // Wrappers (not List<sObject>)
        public List<Wrapper> wrappers { get; private set; }

        public ApexController() {
            List<Contact> cts = [SELECT Id, Name, Account.Name 
                                 FROM Contact 
                                 WHERE LastName LIKE 'Test%'];

            wrappers = new List<Wrapper>();

            for (Contact ct : cts) {
                wrappers.add(new Wrapper(ct));
            }

            List<Lead> leads = [SELECT Id, Name, LeadSource 
                                FROM Lead
                                WHERE LeadSource = 'Web'];

            for (Lead l : leads) {
                wrappers.add(new Wrapper(l));
            }
        }
    }

In your Visualforce page, you can use conditional rendering to select which set of data points to show based on whether you have an `Contact` or a `Lead` inside each wrapper. For example:

    <apex:repeat value="{! wrappers }" var="w">
        <!-- We can dynamically select CSS classes based on what each wrapper contains -->
        <div class="{! IF(w.dataType = 'Lead', 'div-class-lead', 'div-class-contact') }">
            <apex:outputText value="{! w.calculatedTitle }" /><br />
            <apex:outputText rendered="{! w.dataType = 'Lead' }" value="{! 'Lead Source: ' + w.ld.LeadSource }" />
            <apex:outputText rendered="{! w.dataType = 'Contact' }" value="{! 'Account: ' + w.ct.Account.Name }" />
        </div>
    </apex:repeat>

This is a very simple example; you can do quite a bit with conditional rendering to present the wrapper's information most appropriately.

In the case that you're using an `<apex:pageBlockTable>` with `<apex:column>` entries, you might simplify your presentation by creating more calculated properties within your wrapper and keying your columns directly to those `Wrapper` instance variables, rather than using extremely complex conditional rendering. For example, if you were presenting a synthetic Contact "timeline" composed of Activities and Campaign Member records, your wrapper object might consist almost entirely of a set of calculated properties like `Title`, `Date`, and `Description` - you might not even store the sObjects at all!

It's important to bear in mind that processing sObject results into other data structures, like wrapper classes, makes it necessary to [manually enforce CRUD and FLS permissions](https://developer.salesforce.com/page/Enforcing_CRUD_and_FLS). The automatic support provided by Visualforce relies upon using sObjects and fields directly, so wrapper objects that restructure this data come with a requirement to enforce these permissions appropriately.
