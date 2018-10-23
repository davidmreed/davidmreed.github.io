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
        item['URL'] = '/lightning/r/Case/' + item['Id'] + '/view';
    }

Our column entry will end up looking like this:

    {
        label: 'Case Number', 
        fieldName: 'URL',
        type: 'url', 
        typeAttributes: { 
            label: {
                fieldName: 'CaseNumber'
            },
            target: '_self'
        },
        sortable: true 
    }

Then, the result's just what you might think:

![Lightning Data Table]({{ "/public/lightning-datatable-urls/datatable.png" | absolute_url }})

The Case Number column is hyperlinked to open the related Case record.

Note that we're using the Lightning form of the record URL (`/lightning/r/<sObject>/<id>/view`), and we've added the `target: '_self'` attribute to the `typeAttributes` map. This results in the link opening in the current tab, in non-Console applications, and loading a new Console tab in Sales or Service Console. The default behavior, if `target` is not specified, is to open a new browser tab, even in Console applications, which will often not be the desired behavior.

Using the Classic form of the record URL (`/<id>`) does work, but redirects through a translation URL. For future compatibility, it's best to just use the current-generation URL format.

This process of synthesizing new fields for `<lightning:dataTable>` can be repeated arbitrarily, both for URL fields and for other types of calculated data, like icon columns. It's important to remember, of course, that these synthetic properties cannot be persisted to the server because they're not real sObject fields. Input to server actions must be constructed appropriately.

The JavaScript's loosey-goosey type system and object model can be confusing for Apex programmers, but it offers a lot of freedom in return - and the ability to do things with sObjects we'd need a wrapper class to handle in Visualforce.