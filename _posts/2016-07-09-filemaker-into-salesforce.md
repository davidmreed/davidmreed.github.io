---
layout: post
title: FileMaker into Salesforce
---

I've been working on a project to integrate a very important legacy database built in FileMaker ('AM') into Salesforce. 
I decided to perform this integration by building a read-only, one-to-one copy of 
the AM record (the legacy database is not relational) and importing the entire database more or less unchanged. 
This approach ensures full retention of critical historical information, while allowing me to selectively merge 
information with Salesforce contacts using Apex, DemandTools, and Apsona.

As I began building the 'AM Record' object in Salesforce to represent the legacy records, I realized
that the source FileMaker database held even more fields than I thought - over 500. That number
exceeds the limit on custom fields on a Salesforce object, and it also would take forever to build
by hand.

The obvious solution is a script. Unfortunately, FileMaker makes it difficult to extract
information about the database structure in any format that can be manipulated. I didn't want to
export the entire column set and infer typing, both because it's easy to get wrong and because
the database contained roughly 100 calculation and summary fields that did not need to be migrated
to Salesforce. (About 80% of these fields were equivalent to Salesforce rollups or formula
fields, but weren't being used or were calculating based on obsolete data. The other 20%
were providing values that were part of the user interface in the main FileMaker layout).
I wanted to avoid hand-editing the column set in the export file, since I knew I would be
iterating the import several times to get it right. I also couldn't pick and choose in
building the export itself, since the export UI does not identify calculation fields or
display sample data. I needed something that could be parsed and automated.

It's possible to generate an XML schema for a FileMaker database - but only in FileMaker
Pro Advanced. With the Pro edition, my only option was the "Print" button in the Manage Database interface,
yielding a PDF file with columns for Field Name, Field Type, and some additional details on picklist fields
and calculations. I fed this PDF into an online PDF-Excel converter. (There are many; I used Nitro).
It's an ugly solution, but after some hand manipulation to stitch the pages back together, remove
extra rows, and drop out the columns marked as calculations and summaries, I had something
resembling a schema. Final result: 406 fields. Far too many, but reduced enough to fit in one
Salesforce object.

The next step was to map the FileMaker structure into Salesforce's terms. I built a simple Python
script to read the FileMaker schema and spit out XML for insertion into an object definition
in the Force.com IDE. After iterating several times, I discovered that my "schema" didn't
contain enough information on its own to perform the migration, as well as some interesting
features of the Apex Data Loader:

 - "Boolean" columns in FileMaker were actually just Text entries with values of Yes and No.
 - Text columns in FileMaker don't have length limits, as they do in Salesforce.
 - Some of the Text entries were very long (up to 6KB).
 - Apex Data Loader silently truncates overlong Text entries during import operations.
 - Apex Data Loader does understand "Yes" and "No" as legitimate values for importing Checkbox fields.

I got past these issues by revising my script to do some simple type inference, combining
the full data export with my rudimentary FileMaker schema. The final script knows how
to select between a Long Text Area and a Text entry based on the maximum entry length,
and can tell the difference between a Boolean-valued column that should become a Checkbox
and other text entries.

[GenerateObject.py](https://gist.github.com/davidmreed/a7218caf92ab9cf363d4ada9063bab59) is
available if anyone happens to be confronting the same problem. It's not polished, but it works.

Test imports went off with only a couple small hitches: Salesforce bounds the acceptable
values for a Date entry more tightly than FileMaker does, revealing a number of data entry
errors ("0208" for "2008"), and a single column in FileMaker that was typed for date information
but incorrectly contained text. My ad-hoc generation of the schema from FileMaker's PDF file also
inadvertently skipped two fields, which I found and corrected during the process.

Ultimately, I'm pretty satisfied with 85 easily fixable errors out of over 17,000 records.
Having added a Salesforce page layout built to closely approximate the original FileMaker layout,
I'm ready to perform the final import.

In summary, here's the procedure that worked:

1. Create PDF of database spec from the Manage -> Databases window in FileMaker.
2. Convert PDF to Excel to CSV; manually review and correct errors or remove undesired fields.
3. Export entire database (all fields) to Excel and convert to CSV. (I encountered numerous errors while attempting to export from FileMaker directly to CSV).
4. Create stub object in Salesforce and pull metadata using Force.com IDE (or Ant or MavensMate).
5. Run Python script to generate XML field entries for all the columns in the FileMaker schema.
  - Optionally, run the script once with the verification code uncommented and ensure that all discrepancies between the database export and the schema are deliberate.
  - Ensure that generated Salesforce API names are unique, and correct any duplications.
7. Copy and paste XML field entries into the stub object definition in Force.com IDE and push to the server.
8. Use Apex Data Loader to import the exported file into the new object.
   - Automatic mapping worked fine for me, although I manually edited the object's Name field label to be that of the record ID from FileMaker and removed it from the generated schema.
9. Fix any import errors.
