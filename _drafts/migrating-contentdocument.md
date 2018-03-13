---
layout: post
title: Instance-to-Instance Migration with `ContentDocument` 
---

I recently completed an instance-to-instance data migration wherein the source organization used new-style attachments ("Files") pervasively. De

1. Perform a Scheduled Data Export on the source organization, making sure to check the box to include attachments and content files.
1. Use a Python script to iteratively query all of the `ContentDocumentLink` records in the source organization for the exported `ContentVersion` records (which include their `ContentDocumentLinkId` values). `ContentDocumentLink` can only be queried if an `Id` filter is included. For efficiency, the script batches 
1. Optionally, filter on `IsLatest` = 1 to remove old versions. 
1. Construct an import file for the new organization, with a `Body` column referencing the content version's directory and the Id.
1. Import the `ContentVersion` objects into the target organization, which inherently creates `ContentDocument` records.
1. Re-export the ContentVersion objects with their ContentDocumentId column included using Workbench or the Data Loader. Note that you must be logged in as the same user that created the ContentVersions, as the View All Data permission does not apply to private content.
1. Construct a mapping table starting from the `ContentDocumentLink` table exported from the source org. Map the original `ContentDocumentId` to the original `ContentVersionId` (in the original export file) to the new `ContentVersionId` (in your Data Loader success file) to the new `ContentDocumentId` (in the export created in the previous step).
1. Construct a new `ContentDocumentLink` table using the mapped `ContentDocumentId` values and the original sharing values. You'll have to map the `LinkedEntityId` to your new system as well, based upon the remainder of your data migration activity. I like to build a "global ID map" in a single sheet with all of my original Salesforce Ids and new Salesforce Ids, for easy `INDEX/MATCH` lookup.
1. Insert the new `ContentDocumentLink` objects.
