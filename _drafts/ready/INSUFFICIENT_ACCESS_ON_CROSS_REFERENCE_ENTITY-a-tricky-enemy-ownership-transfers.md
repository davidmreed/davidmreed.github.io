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

Similarly, they wouldn't be able to find the original `Lead` by querying against its Id in a Private sharing environment, unless some sharing rule or permission provided them with access, but the programmatic sharing issue is more insidious because Apex retains in-memory sObjects and Ids from the point when the user *did* have access to them.

Because of the multiplicity of routes to Full Access and common tendency to test code as a System Administrator who possesses Modify All Data permission (unless the test class generates its own user), this error class can sneak by unit testing processes and can be tricky to pin down and debug. 

There's a couple of routes to a fix.

One blunt-instrument approach is to run the class `without sharing`. In many circumstances that's fine (or even *required*) for functionality like a trigger handler, but it's a decision that must be made based on the totality of the organization's sharing architecture and visibility needs, as well as the architecture of the specific application.

A finer-instrument fix is to use a small helper class that runs `without sharing`. This could be an inner class, and its only job may be to perform the privileged operation that cannot succeed in the parent class:

    private without sharing class ShareHelper {
        private static void addShares(List<LeadShare> shares) {
            insert shares;
        }
    }

Specific sharing architectures and code structures can raise other possible solutions. In some situations, simply reordering operations to perform privileged actions prior to transferring ownership can be sufficient to resolve the issue. Alternately, adding programmatic shares to the original record owner prior to the transfer can allow privileged operations to succeed. Of course, this must be conformant with the origanization's visibility expectations.
