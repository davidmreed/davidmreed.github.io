---
layout: post
title: "`INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY`, A Tricky Enemy: Ownership Transfers and Sharing Access"
---

Ah, `INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY`, bane of Salesforce developers. The core meaning of this error message is simple: upon performing DML on some record A, there's an issue with record-level sharing associated with one of the records linked to ("cross reference") by A. But the details get very murky. Because Salesforce's record-level sharing architecture takes in so many facets of the organization's configuration and depends so heavily on the transaction context user, debugging this error is often very challenging.

In this post and a sequel, I look at two ways `INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY` can manifest when working with Apex sharing and record ownership. I'll also look at how the sharing declaration of the Apex class - `with sharing`, `without sharing`, or `inherited sharing` - does or doesn't help to cure the problem, and why these issues can be difficult to catch in unit tests.

First, we'll look at a problem class that - while it can be challenging to pin down - is relatively easy to fix. We're working in an organization with a Private sharing model on Lead, and building Apex code that performs Lead assignments programmatically. We'd like, though, for the original Lead owner to retain visibility when the Lead is assigned.

We've come up with a class like this:

	public with sharing class LeadService {
		public static void updateLeadOwnership(List<Lead> newList) {
			List<Lead> toUpdate = new List<Lead>();

			for (Lead l : newList) {
				toUpdate.add(new Lead(Id = l.Id, OwnerId = getNewOwnerForLead(l)));
			}

			update toUpdate;

			List<LeadShare> shares = new List<LeadShare>();

			for (Lead l : newList) {
				shares.add(generateShare(l));
			}

			insert shares; 
		}
	}

At `insert shares`, we'll get

> INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY, insufficient access rights on cross-reference id: []

The critical issue here is that lead ownership is transferred away from the running user during the trigger's operation. Once that update has taken place, based on a Private Organization-Wide Default, the running user no longer has the right to add a `LeadShare` entry (they're not the owner anymore), unless they possess some overriding permission like "Modify All Data" or occupy a position above the new owner in the role hierarchy.

Similarly, they wouldn't be able to find the original `Lead` by querying against its Id in a Private sharing environment, unless some sharing rule or permission provided them with access. The programmatic sharing issue is more insidious because Apex retains in-memory sObjects and Ids from the point when the user *did* have access to them.

Because of the multiplicity of routes to Full Access and common tendency to test code as a System Administrator who possesses Modify All Data permission (unless the test class generates its own user), this error class can sneak by unit testing processes and can be tricky to pin down and debug.

Luckily, this manifestation of `INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY` is pretty easy to fix. It stems from a sort of visibility mismatch — our code's gotten its hands on some information, a Salesforce Id, that the current sharing regime doesn't allow it to see at the database level. We can fix that by running the code in system mode with a `without sharing` declaration.