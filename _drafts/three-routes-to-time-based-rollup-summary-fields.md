---
layout: post
title: Three Routes to Time-Based Rollup Summary Fields
---

Rollup Summary Fields are *great*. Users love them because they offer at-a-glance insight at the top level of the data hierarchy, without needing to run a report or drill down in the user interface to child records. Unfortunately, native Rollup Summary Fields come with a variety of limitations on what data you can roll up, where, and applying which criteria. In particular, *time-based rollup summary fields* are a common need that's tricky to meet with this native functionality.

These fields come from business requirements like this:

> As a Sales representative, I need to see on the Account a running total of Opportunities on a year-to-date and month-to-date basis.

## Naïve Solutions and Why They Don't Work

Most naïve solutions to this class of requirements fail because they're based on formula fields or date literals. One might try creating a formula like this, for example, for a Checkbox field denoting whether or not to include a specific record:

    MONTH(CloseDate) = MONTH(TODAY()) && YEAR(CloseDate) = YEAR(TODAY())

Immediately, though, we find that this new field isn't available for selection in our Rollup Summary Field criteria screen.

Likewise, we can't save a Rollup Summary Field where we write criteria like 

    CloseDate EQUALS THIS_MONTH

using a date literal like we might in a report.

The same underlying limitation gives rise to both of these restrictions: native Rollup Summary Fields require a *trigger event* on the rolled-up object in order to update the rolled-up value. Rollup Summary Fields are evaluated and updated as part of the [Trigger Order of Execution](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_order_of_execution.htm#!) (steps 16 and 17).

Formula fields, though, don't have any underlying storage, and their values are not persisted to the database. Rather, their values are generated at the time the record is viewed. For this reason, there's no event when a formula field's value "changes". In fact, some formula fields can be nondeterministic and incorporate date or time fields, such that their value is essentially always in flux.

> λέγει που Ἡράκλειτος ὅτι ‘πάντα χωρεῖ καὶ οὐδὲν μένει’<br />
> Herakleitos says that 'everything changes, and nothing stands still' <br />
> — Plato, *Cratylos* 402a 

Likewise, there's no trigger event, on any record when the current day changes from one to the next. That's why we cannot embed date literals in our Rollup Summary Field criteria - when today ticks over to tomorrow, not only is there no trigger event, but a recalculate job could be very computationally expensive.

The approaches we will take to solve this requirement ultimately take both tacks: creating a trigger event based on the date, and arranging for broad-based recalculation on a schedule.

## Approach 1: Native Rollup Summary Fields with Time-Based Workflow or Process

Our first approach maximizes use of out-of-the-box functionality, requiring no Apex and no additional packages. However, it can scale poorly and is best suited for smaller Salesforce orgs.

We can't create a native Rollup Summary Field, as we've seen, based on a formula field. But we can rollup based upon a Checkbox field, and we can use Time-Based Workflow and Processes to update that Checkbox on a schedule, using formula fields to get the dates right.

Here's how it works:

 1. We create two Date formula fields on the child object (here, Opportunity). One defines the *date when the record enters rollup scope* and the other the *date when it exits rollup scope*.<br/>
  ![First Day in Rollup Scope]({{ "/public/time-based-rollups/First Day in Rollup Scope.png" | absolute_url }})
  ![Last Day in Rollup Scope]({{ "/public/time-based-rollups/Last Day in Rollup Scope.png" | absolute_url }})
 1. We create a Checkbox field on the child object. This is what our Time-Based Workflow Actions will set and unset, giving us a triggerable event for the Rollup Summary Field.
 1. We create a Rollup Summary Field on the parent object (here, Account). We use the criterion that our Checkbox field is set to True.<br/>
![Rollup Summary Field]({{ "/public/time-based-rollups/Rollup Summary Field.png" | absolute_url }})

 1. We create a Workflow Rule with two Time-Based Actions attached to it.<br/>
![Workflow Rule]({{ "/public/time-based-rollups/Workflow Rule.png" | absolute_url }})

This approach works well for most sorts of time-based rollup requirements. Because it uses formula fields to define when a record enters and exits the rolled-up period, it's not limited to "this month", "this year", and other simple definitions. The time period doesn't even need to be the same for each record!

### Costs and Limitations

This solution, as noted above, is best suited for smaller Salesforce orgs with moderate data volume and small numbers of time-based rollups. Each time-based rollup will consume three custom fields on the child and one Rollup Summary Field on the parent (the limit for which is [25 per object](https://help.salesforce.com/articleView?id=custom_field_allocations.htm&type=5)), as well as a Process or a Workflow Rule with two attached Actions.

Because of the [mechanics of time-based workflow actions](https://help.salesforce.com/articleView?id=workflow_time_action_considerations.htm&type=5), updates made won't roll up instantaneously, like with a trigger. Rather, they'll be on a small delay as the workflow actions are enqueued, batched, and then processed:

> Time-dependent actions aren’t executed independently. They’re grouped into a single batch that starts executing within one hour after the first action enters the batch. 

More important, though, is the platform limitation on throughput of time-based workflow actions. From the same document:

> Salesforce limits the number of time triggers an organization can execute per hour. If an organization exceeds the limits for its Edition, Salesforce defers the execution of the additional time triggers to the next hour. For example, if an Unlimited Edition organization has 1,200 time triggers scheduled to execute between 4:00 PM and 5:00 PM, Salesforce processes 1,000 time triggers between 4:00 PM and 5:00 PM and the remaining 200 time triggers between 5:00 PM and 6:00 PM.

Data volume is therefore the death of this solution. Imagine, for example, that your org is rolling up Activities on a timed basis - say "Activities Logged This Month". Your email-marketing solution logs Tasks for various events, like emails being sent, opened, or links clicked. Your marketing campaigns do well, and around 30,000 emails or related events are logged each month.

That immediately poses us a problem. When the clock ticks over to the first of the month, for example, we suddenly have 30,000 emails that just went out of scope for our "Activities Logged This Month" rollup. Salesforce now begins processing the 30,000 enqueued workflow actions that have accumulated in the queue for this date, but it can only process 1,000 per hour. Worse, the limit is 1,000 per hour *across the whole org* - not per workflow. So we now have a backlog 30 hours deep that impacts both our Rollup Summary Field *and* any other functionality based on timed actions across our org.

If you have a relatively small number of rollups to implement and relatively little data volume, this solution is both effective and expedient. Other organizations should read on for more scalable solutions.

## Approach 2: DLRS in Realtime + Scheduled Mode

[Declarative Lookup Rollup Summaries](https://github.com/afawcett/declarative-lookup-rollup-summaries) can help address many limitations of native Rollup Summary Fields.
DLRS doesn't have inherent support for time-based rollups, but it does have features we can combine to achieve that effect.

Instead of using a native Rollup Summary Field, we start by defining a DLRS Rollup Summary. We can use any SOQL criteria to limit our rolled-up objects, *including* date literals. As a result, we don't need formula fields on the child object - and we also win freedom from some of the other limitations of native Rollup Summary Fields.

Here's how we might configure our putative Opportunity rollup in DLRS:

![DLRS]({{ "/public/time-based-rollups/DLRS.png" | absolute_url }})

To start with, this Rollup Summary will only work partially. It'll yield correct results if we run a full calculate job by clicking `Calculate`. If we configure it to run in `Realtime` mode and deploy DLRS's triggers, we'll see our rollup update as we add and delete Opportunities. 

What won't work, though, is the shift from one time period to the next. On the first of the month, all of our Opportunities from last month are still rolled up - there was no trigger event that would cause them to be reevaluated against our criteria.

With DLRS, rather than using time-based workflow actions to create a trigger event by updating a field, we recalculate the rollup value across every parent record when each time period rolls over. Here, we've scheduled a full recalculation of the rollup for the first day of each month.

![DLRS Scheduler]({{ "/public/time-based-rollups/DLRS Scheduler.png" | absolute_url }})

Because DLRS also deploys triggers dynamically to react to record changes in real time, we get in some sense the best of both worlds: instant updates when we add and change records on a day-to-day basis, with recalculation taking place at time-period boundaries.

### Costs and Limitations

Each DLRS rollup that's run in Scheduled mode consumes a scheduled Apex job, the limit for which is 100 across the org - and that limit encompasses *all* Scheduled Apex, not just DLRS jobs.

Running rollups in real time requires the deployment of dynamically-generated triggers for the parent and child objects. This may challenge deployment processes or even cause issues with existing Apex development in rare cases.

Processing rollup recalculations can be expensive, long-running operations. A full rollup recalculation must touch each and every parent object in your org.

In general, this solution offers the greatest breadth and flexibility, but also demands the greatest awareness of your org's unique challenges and performance characteristics.

## Approach 3: Native Rollup Summary Fields with Scheduled Batch Apex

We can scale up Approach 1 by replacing the limited Time-Based Workflow Actions with a scheduled Batch Apex class. We retain the native Rollup Summary Field based on a Checkbox on the child record, but we don't require the formula fields on the child.

The Batch Apex class we use is very simple: all it does it query for records whose Checkbox value does not match the position of the relevant Date field vis-a-vis the rollup's defined time interval, at the time the scheduled class is executed.

Continuing the example developed above, where we're rolling up Opportunities for the current month only, our batch class's `start()` method would run a query like this:

    SELECT Id, This_Month_Batch__c
    FROM Opportunity
    WHERE (CloseDate = THIS_MONTH AND This_Month_Batch__c = false) 
          OR (CloseDate != THIS_MONTH AND This_Month_Batch__c = true)

Here we just locate those Opportunities that are *not* marked as in-scope but should be (where `CloseDate = THIS_MONTH` - note that we have the freedom to use date literals here), or which *are* marked as in scope but should not be.

Then, the work of our `execute()` method is extremely simple: all it does is reverse the value of the Checkbox field `This_Month_Batch__c` on each record:

    public void execute(Database.BatchableContext bc, List<Opportunity> scope) {
        for (Opportunity o : scope) {
            o.This_Month_Batch__c = !o.This_Month_Batch__c;
        }
        update scope;
    }

We'd schedule this batch class to run every night after midnight. When executed, it'll update any records moving into or out of rollup scope that day, allowing the native Rollup Summary Field machinery to do the work of recalculating the Account-level totals.

### Costs and Limitations

While it's not Apex-intensive, this is the only solution of the three that is not declarative in nature, and all of the attendant costs of code-based solutions are present.

Organizations with high data volume or data skew may need to experiment carefully to ensure queries are properly optimized.

As with DLRS, each rollup using this strategy consumes one scheduled Apex job. Unlike DLRS, we need to run our scheduled class nightly, because we're not combining it with a trigger for (partial) real-time operation. We could add more Apex to do so if desired.