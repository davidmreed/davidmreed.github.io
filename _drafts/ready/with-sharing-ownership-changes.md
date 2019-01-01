---
layout: post
title: `INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY`, A Tricky Enemy: Ownership Transfers and Sharing Access
---

There's a set of sharing configurations that can cause very interesting - and tricky to debug - access exceptions in Apex. They share in common the following features:

 1. The code is running `with sharing`, often in a trigger handler, or doesn't declare a sharing model and inherits `with sharing` from its caller.
 2. The code is operating on an object whose Organization-Wide Default is Private, or in some situations Public Read Only.
 3. The code performs an ownership transfer as well some other privileged operation, such a read/write operation or programmatic sharing.
 4. Usually, the code is affecting a standard object (i.e., *not* using Apex Managed Sharing).

An affected solution tends to look like this, albeit often more complex:

	public static void afterInsert(List<Lead> newList) {
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
		// Exception is thrown! INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY, insufficient access rights on cross-reference id: []
	}

The critical issue is that lead ownership is transferred away from the running user, assuming they're working on their own records, during the trigger's operation. Once that update has taken place, based on a Private Organization-Wide Default, the running user no longer has the right to add a `LeadShare` entry in the `after update` event (they're not the owner anymore), unless they possess some overriding permission like "Modify All Data" or occupy a position above the new owner in the role hierarchy. 

Similarly, they wouldn't be able to find the original `Lead` by querying against its Id in a Private sharing environment, unless some sharing rule or permission provided them with access, but the programmatic sharing issue is more insidious because Apex retains in-memory sObjects and Ids from the point when the user *did* have access to them.

Because of the multiplicity of routes to Full Access and common tendency to test code as a System Administrator who possesses Modify All Data permission (unless the test class generates its own user), this error class can sneak by unit testing processes and can be tricky to pin down and debug. 

There's a couple of routes to a fix.

One blunt-instrument approach is to run the trigger handler `without sharing`. In many circumstances that's fine (or even *required*) for a trigger handler, but it's a decision that must be made based on the totality of the organization's sharing architecture and visibility needs.

A finer-instrument fix is to use a small helper class that runs `without sharing`. This could be an inner class of the trigger handler, and its only job would be to perform the privileged operation that cannot succeed in the parent trigger handler:

    private without sharing class ShareHelper {
        private static void addShares(List<LeadShare> shares) {
            insert shares;
        }
    }

Specific sharing architectures and code structures can raise other possible solutions. In some situations, simply reordering operations to perform privileged actions prior to transferring ownership can be sufficient to resolve the issue. Alternately, adding programmatic shares to the trigger's running user/the original record owner prior to the transfer can allow privileged operations to succeed. Of course, this must be conformant with the origanization's visibility expectations.
