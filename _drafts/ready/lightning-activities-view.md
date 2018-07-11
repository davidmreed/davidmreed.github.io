---
layout: post
title: Filtering in Lightning's Activity Timeline 
---

The Lightning Experience's record pages come with a very nice Activity timeline and publisher.

![Lightning Activity Timeline]({{ "/public/lightning-activities-view/lightning-activities-view.png" | absolute_url }})

One of the features of this timeline is highlighting different types of activities with topical icons, and permitting the user to apply filters on the fly to isolate activities of interest - like just Emails, or just Calls, or all Tasks owned by the user.

![Filters for Activity Timeline]({{ "/public/lightning-activities-view/lightning-activities-filter.png" | absolute_url }})

How, though, does Lightning distinguish between these different activities to populate the various filters? And further, can we control that filtration to place activities we generate under specific filter headings?

The answers turn out to be "a bit of magic" and "sort of." Lightning recognizes five categories of activity: Emails, Events, List Emails, Logged Calls, and Tasks.

Events are distinguished from Emails, List Emails, Logged Calls, and Tasks by sObject type. Events have the object type `Event`, while the other four are all of type `Task`. The Emails, List Emails, Logged Calls, and Tasks filters work differently. All of them filter based upon the picklist value in the field `TaskSubtype` on the `Task` object. 

Confusingly, this field isn't connected to the `Type` field whatsoever - it's not a dependent field, and the `Type` field plays no role in Activity filtering.

This field is populated by the system, and cannot be changed. At the database level, it's [createable but not updateable](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_objects_task.htm). 

> \[`TaskSubtype`\] Provides standard subtypes to facilitate creating and searching for specific task subtypes. This field isn’t updateable.

The four values of this picklist (which is restricted) are `Email`, `ListEmail`, `Call`, and `Task`, each of which maps to exactly one filter. `Email` and `ListEmail` are entirely separate; they don't appear in one another's filtered view.

## Manipulating the `TaskSubtype`

Knowing, then, how Lightning sorts tasks under different filter headings, can we manipulate the filters to sort our custom activities in bespoke ways? Only to a slight extent, as it turns out.

We cannot change the behavior of the filters. The Activity timeline is a non-configurable Lightning component; we can add it to record pages, but it doesn't have any public attributes for us to set.

Because the `TaskSubtype` field is createable, but not updateable, we cannot move existing records from one category to another.

The one route we have is to override the category at the time of creation of the `Task`, and it works only in one situation. If the `TaskSubtype` is ultimately going to be `'Task'` (i.e., it is not an Email, List Email, or Call), we can, in a `before insert` trigger on `Task`, set the `TaskSubtype` to one of the other three values. A putative Task can be transmogrified into an Email, Call, or List Email:

![Task converted to Call]({{ "/public/lightning-activities-view/task-converted-to-call.png" | absolute_url }})

However, this does not work in the other direction. Records that are being created as Emails, List Emails, or Calls can't have their `TaskSubtype` set to `'Task'`, or to any of the other available values. Attempting to do so simply has no effect, although it doesn't cause an error. 

Additionally, `before insert` trigger code cannot introspect the value of the `TaskSubtype` field in any logic, because it is not populated by the system until the `after insert` context - where it cannot be altered. Since it doesn't appear to be inferred from any values that *are* populated at `before insert` time, it seems to simply be magic.

There's still one more caveat of applying this technique, though: if the Task-to-be-converted is added via the interactive publisher, Chatter records the original type of the task in its feed:

![Chatter post]({{ "/public/lightning-activities-view/converted-task-record-chatter.png" | absolute_url }})

This mismatch does not occur when the `Task` is inserted via code, which doesn’t produce a Chatter post and hence preserves the illusion of being (say) a Call the entire time.

The question of whether it's *wise* to manipulate the Activity timeline in this way is another story. Because this functionality is at least somewhat undocumented, I wouldn't rely on it continuing to work exactly the same way in future API versions. Vote for [this Idea](https://success.salesforce.com/ideaView?id=0873A000000COq5QAG) to make `TaskSubtype` editable!

One last interesting facet: there's similar data model on `Event`, with [`Event.EventSubtype`](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_objects_event.htm). Like with `Task`, this field isn’t updateable. However, permitted values are not documented, and the picklist has only a single value, 'Event', which is populated on standard events. Perhaps we'll see more functionality around timeline filtering in future releases.
