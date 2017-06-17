---
layout: post
title: Visualforce Traps for Experienced Programmers
---

Visualforce has a way of punishing those who make assumptions about its behavior based on other development environments (including Apex and even HTML). I've been accumulating a list of Visualforce behaviors and features that seem inexplicable to me at first blush, or that I've tripped up on multiple times. Whenever I'm debugging, I check this list of mistakes first. Suggested additions are welcome!

## Parameters Not Being Set

The `<apex:param>` component silently does not set its parameter unless the `name` attribute is populated, although omitting `name` is not an error, and the attribute doesn't make sense for command button parameters.

## Race Condition / Incorrect Serialization of Actions

It's common to need to set a value in the controller and subsequently trigger a partial page re-render. In some cases, there's no controller action required at all; in others, the business logic may be embedded in a controller setter method rather than a Visualforce action.

The naïve way to handle this is an `<apex:commandButton>` with one or more `<apex:param>` elements and a `reRender`, but no `action` set. Unfortunately, this causes a difficult-to-debug race condition. The server calls that set the parameters and perform the re-render are not serialized, meaning that the re-render will be working with stale data some, but not all, of the time.

Workaround: an empty action method in the controller. This forces a complete server round-trip prior to beginning the re-render.

## Required Non-sObject Fields

It seems obvious to use the `required` attribute on, for example, `<apex:inputText>` components to obtain the standard Salesforce UI presentation of a red bar beside the component, when working with non-sObject fields. Unfortunately, this doesn't work. Instead, you can use a [workaround](https://salesforce.stackexchange.com/questions/2108/style-apexinputtext-as-required-within-apexpageblocksectionitem) with a styled empty `<div>`.

## Misplaced Labels
`<apex:outputLabel>` is position-dependent. In order to be rendered with the correct style as part of an `<apex:pageBlockSectionItem>`, it must precede its `for` element, and not be nested within that element.

    <apex:pageBlockSectionItem>
      <apex:outputLabel for="outValue" value="A Label" />
      <apex:outputText value="{! someControllerValue }" id="outValue" />
    </apex:pageBlockSectionItem>

The above shows the correct ordering to receive standard Salesforce styling.

## `reRender` Targets and the `rendered` Attribute

Showing and hiding page elements based on changes to controller values is a very common workflow. It's easy to forget that you cannot do this by setting the `rendered` attribute on a target and then performing a `reRender` on that very same target. Since the elements with `rendered="false"` are never even sent to the client, they can't be re-rendered in the normal way.

Instead, wrap the element with the `rendered` attribute in an `<apex:outputPanel>` that you `reRender` to show and hide its content.

## Accessing Visualforce Components in JavaScript.

Accessing Visualforce components in JavaScript using `document.getElementById()` with `$Component` is highly unintuitive. `id` values assigned to Visualforce components aren't global like they are in HTML, so in many cases you need to qualify them in order to obtain a reference to the component. More confusingly, the manner in which you must so qualify the component reference is partially dependent upon where in the page the JavaScript is located.

Salesforce has several pages that attempt to explain this, but by far the most useful is this [set of examples](https://developer.salesforce.com/docs/atlas.en-us.pages.meta/pages/pages_best_practices_accessing_id.htm).

Where else is Visualforce likely to trip up coders trying to apply experience from other environments?  
