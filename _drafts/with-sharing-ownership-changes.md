---
layout: post
title: Trigger Handlers, `without sharing`, and Ownership Transfers
---

There's a set of sharing configurations that can cause very interesting - and tricky to debug - access exceptions in Apex. They share in common the following features:

 1. The code is running `with sharing`, often in a trigger handler, or doesn't declare a sharing model and inherits `with sharing` from its caller.
 2. The code is operating on an object whose Organization-Wide Default is Private.
 3. The code performs an ownership transfer as well some other privileged operation, such a read/write operation or programmatic sharing.
 4. Usually, the code is affecting a standard object (i.e., *not* using Apex Managed Sharing).

An affected solution tends to look like this:

    public with sharing class LeadTriggerHandler {
        public static void afterInsert(List<Lead> newList) {
            for (Lead l : newList) {
                l.OwnerId = getNewOwnerForLead(l);
            }

            update newList;

            List<LeadShare> shares = new List<LeadShare>();

            for (Lead l : newList) {
                shares.add(generateShareToOldOwner(l));
            }

            insert shares; // Exception is thrown!
        }
    }

The critical issue is that lead ownership is transferred away from the running user during the trigger's operation. One that update has taken place, based on a Private Organization-Wide Default, the running user no longer has the right to add a `LeadShare` entry in the `after update` event (they're not the owner anymore), unless they possess some overriding permission like "Modify All Data" or occupy a position above the new owner in the role hierarchy. Similarly, they wouldn't be able to find the original `Lead` by querying against its Id in a Private sharing environment, unless some sharing rule or permission provided them with access.

Because of the multiplicity of routes to Full Access and common tendency to test code as a System Administrator who possesses Modify All Data permission, this error class can sneak by unit testing processes and can be tricky to pin down and debug. 

There's a couple of routes to a fix.

The blunt-instrument approach is to run the trigger handler `without sharing`. In many circumstances that's fine (or required) for a trigger handler, but it's a decision that must be made based on the totality of the organization's sharing architecture and visibility needs.

One finer-instrument fix is to use a small helper class that runs `without sharing`. This could be an inner class of the trigger handler, and its only job would be to perform the privileged operation that cannot succeed in the parent trigger:

    private without sharing class ShareHelper {
        private static void addShares(List<LeadShare> shares) {
            insert shares;
        }
    }

Specific sharing architectures can raise other possible solutions. As one example, if it's desirable for the original owner to retain view permission on the transferred object (or if this visibility )