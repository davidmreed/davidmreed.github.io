---
layout: post
title: Handling URLs in `<lightning:dataTable>`
---

The [`<lightning:dataTable>`](https://developer.salesforce.com/docs/component-library/bundle/lightning:datatable) component has built-in support for displaying links in table columns. The syntax looks something like this:

    {
        label: 'Case Number', 
        fieldName: 'My_URL_Field__c',
        type: 'url', 
        typeAttributes: { 
            label: {
                fieldName: 'CaseNumber'
            } 
        },
        sortable: true 
    }

`typeAttributes.label.fieldName` identifies a field on each row to utilize as the title of the link, while `fieldName` at the top level specifies the URL field itself.

In many cases, though, what we have in our sObject data isn't a fully-qualified URL: it's a Salesforce Id, a lookup to this record or to some other record, and we'd really like to display it sensibly as a link with an appropriate title. Unfortunately, `<lightning:dataTable>` doesn't have an `Id` column type, and the `url` type is not clever enough to realize it's been handed a record Id and handle it.

Instead, we need to generate the URL ourselves and add it as a property of the line items in the source data. (This is a bewildering shift for seasoned Apex programmers: *we can just add fields to our sObjects?!*) In the callback from the Apex server method querying our sObjects, we generate one or more synthetic properties:

    cases.forEach(function(item) {
        item['URL'] = '/' + item['Id'];
    }

Your column entry will end up looking like this:

    {
        label: 'Case Number', 
        fieldName: 'URL',
        type: 'url', 
        typeAttributes: { 
            label: {
                fieldName: 'CaseNumber'
            } 
        },
        sortable: true 
    }

Then, the result's just what you might think:

This process can be repeated arbitrarily, both for URL fields and for other types of synthesized data intended for the `<lightning:dataTable>`.

It's important to remember, of course, that these synthetic properties cannot be persisted to the server because they're not real sObject fields. Input to server actions must be constructed appropriately.