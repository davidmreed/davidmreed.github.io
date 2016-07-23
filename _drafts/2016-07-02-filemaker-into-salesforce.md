---
layout: post
title: FileMaker into Salesforce (Part 1)
---

One of the largest challenges involved in the migration to Salesforce that I've been working
on for six months (and expect to continue for some time) is the *other* legacy database. While the primary migration
was from The Raiser's Edge, a second product ('AM'), built in FileMaker, also provided CRM functionality
in areas that weren't well served by Raiser's Edge. The two databases were never integrated, since
RE wasn't configurable enough to meet requirements and AM, a much older solution, had the force of inertia behind it.

Salesforce offers enough configurability and a strong enough toolkit that I can now approach the
challenge of integrating AM, its vast corpus of historical data, and all of the processes built on it
into Salesforce. This will be the first of several discussions of the integration effort.

## First Steps

AM is not relational. While mostly this is a negative (the data is inherently relational and was shoehorned into a
flat file over the course of fifteen years), it allows me to take an incremental approach that addresses some of
the unique issues with this database:

 - The contents are uniquely valuable, including data on board membership and core organizational activities.
 - It is still in use by departments that haven't been brought in to Salesforce yet.
 - Because of its longevity, certain processes are built around the database and staff are comfortable with how it works.
 - Also because of its longevity, there are fields in AM the meaning of which no current staff understand.
 - There is concern about a transition process losing valuable data: even some of the not-presently-understood fields
   record information that may be deemed valuable in the future concerning the details of past programs.

For these reasons, I thought it best to preserve an archival copy of each record as the data (that portion which is well
structured and understood) is integrated into Salesforce records. I'd initially thought to do so by attaching a Note
to each Contact record or by creating a Long Text Area field that would hold an XML or JSON representation of the original
data, while attempting to merge major portions of the data, such as contact information and event histories, into
the existing Contacts.

Since the data in AM is so poorly structured, and because that database existing alongside Raiser's Edge for so
long while serving many of the same constituent relationships, I anticipated writing a lot of Python to perform
conditional merges and handle thousands of merge conflicts. Many of the conflicts would likely be irreconcilable,
since neither legacy database contained enough version history to determine which data were more reliable or up to date.
It was looking like a drawn-out mess that would take me months (that I didn't have) and potentially require me to make
choices about discarding conflicted data that I wasn't comfortable with. I'd also have to choose between creating
hundreds of custom fields on the Contact record to hold historical data, or relegating them to the archive
and creating a need for complex retrieval processes if that information ever became important.

I realized that there was a much easier solution: because AM wasn't relational, I could simply create a custom object
that was a one-to-one copy of the AM record. I'd perform the import, after a light data cleaning pass, into that object,
and use Apex code to connect AM records to matching Contacts (or create missing Contacts). I'd then be at liberty to carry
over data from AM records into Contacts incrementally, using Apex, Apsona, and DemandTools, while providing staff with a
permanent, read-only, fully accessible copy of the data they're used to seeing.
