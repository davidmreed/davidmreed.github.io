The "Transfer Lead" permission is not the issue here, nor the users' Profiles. Rather, it's that once lead ownership is transferred away from the running user, based on your Private Organization-Wide Default, the running user no longer has the right to add a `LeadShare` entry in the `after update` event (they're not the owner anymore). If, as I assume is the case, your trigger handler is running `with sharing`, that OWD is going to be enforced and result in this exception.

The blunt-instrument fix is to run your trigger handler `without sharing`. In many circumstances that's fine for a trigger handler, but only you can determine whether it's appropriate here. 

The finer-instrument fix is to use a small helper class that runs `without sharing`. This can be an inner class of your trigger handler, and its only job would be to add sharing records:

    private without sharing class ShareHelper {
        private static void addShares(List<LeadShare> shares) {
            // Do stuff
        }
    }

I'm not sure I follow the logic of adding share records to the *current* owner of the Lead. That person naturally has edit rights based on their ownership. Is the overall objective to maintain something like a "Lead Team", and retain all previous owners' edit access on the Lead? Your current code isn't looking at the pre-update values of `OwnerId`, so it's only trying adding shares for the current owner.