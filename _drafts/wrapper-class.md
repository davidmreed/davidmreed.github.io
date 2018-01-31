---
layout: post
title: Everyday Salesforce Patterns: The Wrapper Class 
---

It's extremely common in Visualforce development to need to perform some transformation on data queried out of Salesforce before displaying that data. This can include things like enriching one object with data from another (which is not its parent or child), 'framing' or presenting multiple unrelated objects in a single flat list, such as an `<apex:pageBlockTable>`, applying a mapping table to values in an object's fields, or appending summary data calculated in Apex.

An apt solution to all of these needs is the *wrapper class* pattern. Every wrapper class looks a little different, because it's highly specific to the individual use case and Visualforce page. The overall pattern often looks rather like this example, which wraps either an Account or a Lead in an inner class, called `Wrapper`, of the page controller. Wrapper classes do not have to be inner classes, and larger, more complex wrappers in particular may be independent, but the use of an inner class is common and effective.


	public with sharing class ApexController {
		public class Wrapper implements Comparable {
		    Account act;
		    Lead ld;
		    String dataType; // Included for easier conditional rendering.
		    String calculatedTitle; // and so on...

		    public Wrapper(Account a) {
		    	act = a;
		    	dataType = 'Account';
		    	calculatedTitle = 'Account: ' + a.Name;
		    }

		    public Wrapper(Lead l) {
		    	ld = l;
		    	dataType = 'Account';
		    	calculatedTitle = 'Lead: ' + l.Name;
		    }

		    public Integer compareTo(Object other) {
		    	Wrapper o = (Wrapper)other;

		    	// Perform some comparison logic here, such as comparing the `SystemModStamp`
		    	// of the embedded sObjects, or comparing the calculatedTitle properties.

		        return 0;
		    }
		}

		// Declare our public property for Visualforce as a list of Wrappers (not List<sObject>)
		public List<Wrapper> wrappers { get; private set; }

		public ApexController() {
			List<Account> acts = [SELECT Id, Name FROM Account WHERE Name LIKE 'Test%']0;

			for (Account act : acts) {
				wrappers.add(new Wrapper(act));
			}

			List<Lead> leads = [SELECT Id, Name FROM Lead WHERE LeadSource = 'Web'];

			for (Lead l : leads) {
				wrappers.add(new Wrapper(l));
			}
		}
	}


		In your Visualforce page, you can use conditional rendering to select which set of data points to show based on whether you have an EmailMessage or a CaseComment inside each wrapper.

		You can extend the pattern and make your life easier in several ways:

		    You can copy the date of each object into your wrapper, to simplify sorting code.
		    You can go further and populate a chosen set of values from each object into your wrapper directly, to simplify your Visualforce and reduce the amount of conditional rendering. So you might have a title, subject, and a date field on your wrapper object you populate directly in Apex.
		    You might add constructors to your wrapper class that take an EmailMessage or a CaseComment and populate fields appropriately.
