---
layout: post
title: Importing Notes in Salesforce
---

Building Note records in Salesforce, using the new notes introduced in Winter '16, is challenging for a number of reasons. They come with unusual data-preparation requirements and are [tricky to import](https://help.salesforce.com/apex/HTViewSolution?id=000230867&language=en_US) using the Data Loader. Notes are tricky to create in Apex for largely the same reasons. Failing to follow the requirements for encoding incoming data typically produces notes with all of the line breaks omitted and/or unpredictable and difficult-to-debug exceptions.

To make matters worse, various documentation entries are either incorrect (the [API reference on `ContentNote`](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_objects_contentnote.htm) specifies to use `String.escapeHTML4()` to prepare content, which doesn't work) or overly vague and/or incomplete (the `String` API documentation on `escapeHTML4()` and `escapeXML()`, the `ContentNote` import instructions).

In summary, here's what is actually required to insert `ContentNote`s:

 1. Replace all basic HTML characters (`<>"'&`) with their corresponding entities (`&amp;` and friends).
 2. Replace all line breaks with `<br>` (taking care with Windows CRLF/Linux LF/Mac CR)
 3. Replace `&apos;` with `&#39;`.
 4. Do *not* replace Unicode characters with entities. Other entities, including `&apos;`, result in an exception. Unicode should be left as the bare characters.
 5. Ensure that the source content is well-formed Unicode/UTF-8 and does not contain non-printable characters.
 6. The title must not be `null`, zero-length, or consist only of whitespace. The title need not be escaped.

This 4th point is why `String.escapeHTML4()` doesn't work for preparing note text: this method replaces Unicode characters with HTML entities, which causes exceptions upon insert. `String.escapeXML()` is closest to what is needed, but doesn't handle item 3 above.

In frustration with all the hoops involved, I put together a package that can reliably import and add notes and attachments, both programmatically and in bulk. The package provides a `DMRNoteAttachmentImporter` class that can be used in Apex code to create note and attachment records either singly or in bulk, as well as a Note Proxy object that can be converted into Notes using a batch process. Notes may be imported to Note Proxy from a CSV file using any data loader (obviating the one-file-per-note requirement imposed by Apex Data Loader).

Attachments are handled alongside Notes, since the machinery for creating them is very similar.

The code is MIT licensed. It will fail if the new notes have not been [enabled](http://releasenotes.docs.salesforce.com/en-us/winter16/release-notes/notes_admin_setup.htm) in your Salesforce instance. It's been tested with both Apex unit tests (100% coverage) and example files. Bug reports and patches are welcomed.

Be aware that, using the Note Proxy object, you can only import notes of lengths up to the limit of a Long Text Area, 131,072 characters (128KB, assuming 1-byte characters). In my testing, it's fine to process 32KB notes in batches of 200. Stepping up to 64KB notes (which some spreadsheet applications and data loaders cannot process in CSV format in any case) required a reduction in batch size.

Errors that do occur with `ContentNote`s usually come in the form of `System.UnexpectedException`s, which cannot be caught or handled. Despite the best efforts of this package, it is still possible to trigger these exceptions by attempting to import notes whose text contains non-printable characters or mangled UTF-8. In a bulk note import process, this will cause the failure of the entire batch with no error message recorded on the note proxies. The error can be diagnosed by examining the Apex Jobs log, where the exception will be displayed. The only workaround is to use very small (or even 1) batch sizes to identify the offending note and manually correct its text.

## Resources

- [DMRNoteAttachmentImporter](https://github.com/davidmreed/DMRNoteAttachmentImporter) on GitHub.
- [Set Up Notes](http://releasenotes.docs.salesforce.com/en-us/winter16/release-notes/notes_admin_setup.htm)
- [Importing Notes](https://help.salesforce.com/apex/HTViewSolution?id=000230867&language=en_US).
  This reference is correct, if not as complete as might be hoped.
- [ContentNote insert error: ...](https://help.salesforce.com/apex/HTViewSolution?urlname=ContentNote-insert-error-Note-Can-t-Be-Saved-Because-It-Contains-HTML-Tags-Or-Unescaped-Characters-That-Are-Not-Allowed-In-A-Note&language=en_US)
