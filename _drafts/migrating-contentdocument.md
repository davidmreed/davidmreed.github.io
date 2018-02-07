---
layout: post
title: Instance-to-Instance Migration with `ContentDocument` 
---

1. Perform a Scheduled Data Export on the source organization, making sure to check the box to include attachments and content files.
1. Use a Python script to iteratively query all of the `ContentDocumentLink` records in the source organization for the exported `ContentVersion` records (which include their `ContentDocumentLinkId` values). `ContentDocumentLink` can only be queried if an `Id` filter is included.
1. Optionally, filter on `IsLatest` = 1 to remove old versions. 
1. Construct an import file for the new organization, with a Body column referencing the content version's directory and the Id.
1. Import the `ContentVersion` objects into the target organization, which inherently creates `ContentDocument` records.
1. Re-export the ContentVersion objects with their ContentDocumentId column included using Workbench or a similar tool. Note that you must be logged in as the same user that created the ContentVersions, as the View All Data permission does not apply to content.
1. Construct a mapping table starting from the ContentDocumentLink table exported from the source org. Map the original ContentDocumentId to the original ContentVersionId to the new ContentVersionId to the new ContentDocumentId.
1. Construct a new ContentDocumentLink table using the mapped ContentDocumentId values and the original sharing values. You'll have to map the LinkedEntityId to your new system as well, based upon the remainder of your data migration activity.
1. Insert the new ContentDocumentLink objects.
