---
layout: post
title: Everyday Salesforce Patterns: The Wrapper Class 
---

It's extremely common in Visualforce development to need to perform some transformation on data queried out of Salesforce before displaying that data. This can include things like enriching one object with data from another (which is not its parent or child), 'framing' or presenting multiple unrelated objects in a single flat list, such as an `<apex:pageBlockTable>`, applying a mapping table to values in an object's fields, or appending summary data calculated in Apex.

An apt solution to all of these needs is the *wrapper class* pattern. Every wrapper class looks a little different, because it's highly specific to the individual use case and Visualforce page. The overall pattern often looks rather like this example, which wraps either a Contact or a Lead in an inner class, called `Wrapper`, of the page controller. Wrapper classes do not have to be inner classes, and larger, more complex wrappers in particular may be independent, but the use of an inner class is common and effective.

Wrapper classes are in some ways similar to *structures*, *union types*, or *algebraic types* provided by other languages, to the extent such patterns are possible with the limited introspection and dynamism available in Apex.

	public with sharing class ApexController {
      public class Wrapper implements Comparable {
          Lead ld;
          Contact ct;
          String dataType; // Included for easier conditional rendering.
          String calculatedTitle; // and so on... include more calculated fields here.

          public Wrapper(Lead l) {
              ld = l;
              dataType = 'Lead';
              calculatedTitle = 'Lead: ' + ld.Name; // Note that we are assuming the Name field is queried.
          }

          public Wrapper(Contact c) {
            ct = c;
            dataType = 'Contact';
            calculatedTitle = 'Contact: ' + ct.Name;
          }

          public Integer compareTo(Object other) {
              Wrapper o = (Wrapper)other;

              // Perform some comparison logic here, such as comparing the `SystemModStamp`
              // of the embedded sObjects, or comparing the calculatedTitle properties.
              // You can even sort by type by inspecting dataType field, or,
              // if storing sObject instances, with their `sobjectType` field.

              return 0;
          }
      }

      // Declare our public property for Visualforce as a list of Wrappers (not List<sObject>)
      public List<Wrapper> wrappers { get; private set; }

      public ApexController() {
          List<Contact> cts = [SELECT Id, Name, Account.Name FROM Contact WHERE LastName LIKE 'Test%']0;
          
          wrappers = new List<Wrapper>();
          
          for (Contact ct : cts) {
              wrappers.add(new Wrapper(ct));
          }

          List<Lead> leads = [SELECT Id, Name, LeadSource FROM Lead WHERE LeadSource = 'Web'];

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

In the case that you're using an `<apex:pageBlockTable>` with `<apex:column>` entries, you might simplify your presentation by creating more calculated properties within your wrapper and keying your columns directly to those `Wrapper` instance variables, rather than using extremely complex conditional rendering.
