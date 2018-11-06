---
layout: post
title: Running Reports as Selected Users with JWT and the Reports and Dashboards API
---

Salesforce reporting introduces some fascinating complexities to data visibility and exposure, particularly for organizations using Private Organization-Wide Defaults. 

The key complicating factor is this: when a Salesforce report is run, it's run in the context of some user or another, and the records that are shown on the report are the ones that are visible to that user. This means that report *distribution* solutions have to be very careful to only show each user a report run in their own context - not someone else's.

Suppose your organization builds a critical report that many users will need to review. It's built to show "My Opportunities", so each user will see only their own Opportunities, and the Opportunity Organization-Wide Default is Private. You add a criterion to the report to only show Opportunities that have your internal "Needs Attention" Checkbox set. Now: how do you make sure your users are regularly updated when they have Opportunities that require their review?

A naive solution would create one subscription to this report, say for Frank Q. Exec, and add all of the users who need to receive it as recipients:

FIXME: image here

But this runs afoul of the principle mentioned above: the report's context user is Frank, and the recipients of the report will see data *as if they were Frank*. From [Salesforce](https://help.salesforce.com/articleView?id=reports_subscribe_lex.htm&type=5):

> IMPORTANT Recipients see emailed report data as the person running the report. Consider that they may see more or less data than they normally see in Salesforce.

This is unlikely to be an acceptable outcome. 

Further, we can't simply have Frank create many subscriptions to the same report, adding one user as both the recipient and the running user to each: Frank only gets five total report subscriptions, and he can only have one subscription to each report.

Of course, users can schedule reports themselves, in their own context, and they can run them manually, and we can build dynamic dashboards (which come with their own limits). But what if we really need to create these subscriptions for our users automatically, or allow our admins to manage them for thousands of users at a time? What if, in fact, we want to offer the users a bespoke user interface to let them select subscriptions to standard corporate reports, or run reports in their contexts to feed into an external reporting or business intelligence solution?

This is a question I've [struggled with before](https://salesforce.stackexchange.com/questions/202952/subscribe-users-to-reports-run-as-themselves-using-api-or-apex), and I was excited to see Martin Borthiry propose the issue on [Salesforce Stack Exchange](https://salesforce.stackexchange.com/questions/237526/schedule-and-run-a-report-as-an-specific-user-from-apex). Here, I'd like to expand on the solution I sketched out in response to Martin's question. 

## Background

There are two report subscription functionalities on Salesforce, and they work rather differently. Report subscriptions are summarized in the Salesforce documentation under [Schedule and Subscribe to Reports](https://help.salesforce.com/articleView?id=reports_subscribe_overview.htm&type=0).

On Classic, one can "Subscribe" to a report, and one can "Schedule Future Runs". The nomenclature here is confusing: a Classic "Subscribe" asks Salesforce to notify us if the report's results meet certain thresholds, but it's *not* for regularly receiving copies of the report. We're not going to look at this feature. "Schedule Future Runs" is equivalent to a report subscription in Lightning and is the feature corresponding to the business problem discussed above.

FIXME: image here

On Lightning, we simply have an option to Subscribe. There's no Lightning equivalent to the Classic "Subscribe" feature.

FIXME: image here

So what happens when we subscribe to a report?

The Classic Schedule Future Runs and the Lightning Subscribe functionality is represented under the hood as [`CronTrigger`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_crontrigger.htm) and [`CronJobDetail`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_cronjobdetail.htm) records with the `CronJobDetail.JobType` field set to `'A'`, for Analytics Notification. You can find them in queries from the Developer Console or Workbench via queries like

    SELECT CronExpression, OwnerId, CronJobDetail.Name FROM CronTrigger WHERE CronJobDetail.JobType = 'A'

Unfortunately, knowing this doesn't help us very much. Neither `CronTrigger` nor `CronJobDetail` can be created directly in Apex or via the API, and the objects provide very little detail about existing report subscriptions. The Report Id, for example, is notable by its absence, and the `Name` field is just a UUID.

A more promising avenue for our use case is the Reports and Dashboards API, because it offers an [endpoint](https://developer.salesforce.com/docs/atlas.en-us.api_analytics.meta/api_analytics/analytics_api_notification_example_post_notifications.htm) to create an Analytics Notification. 

    POST /services/data/vXX.0/analytics/notifications

with a JSON body like this 

    {
      "active" : true,
      "createdDate" : "",
      "deactivateOnTrigger" : false,
      "id" : "",
      "lastModifiedDate" : "",
      "name" : "New Notification",
      "recordId" : "00OXXXXXXXXXXXXXXX",
      "schedule" : {
        "details" : {
          "time" : 3
        },
        "frequency" : "daily"
      },
      "source" : "lightningReportSubscribe",
      "thresholds" : [ {
        "actions" : [ {
          "configuration" : {
            "recipients" : [ ]
          },
          "type" : "sendEmail"
        } ],
        "conditions" : null,
        "type" : "always"
      } ]
    }
    
The feature set shown here in JSON is at parity with the user interface, and has the same limitations. Adding a recipient for the subscription over the API, for example, suffers from the same visibility flaws as doing so in the UI. And the API doesn't let us do what we truly want to - create report subscriptions *for* other users that run *as* those other users - because we cannot set the owner of the subscription programmatically.

... or can we?

Since the Reporting and Analytics API doesn't support setting the context user for the subscription, we're stuck with controlling the user as whom we authenticate to the API.

With 1,000+ users to schedule, creating individual Named Credentials to authenticate as each user back into the same instance does not seem like a fruitful approach. Instead, I'm suggesting the one route of which I'm aware that does not require each user to individually authenticate and approve access: the JWT OAuth flow. I'll rough out below what this architecture would look like, using an external script to authenticate and update report subscriptions in Salesforce. 

I'm quite honestly unsure of whether you could do this in pure Apex or not; I've never tried to implement JWT from scratch. I would certainly be concerned about secret storage if attempting to build this all on-platform.

## On Platform: UI and Data Model

First, though, the UI side. Have your Lightning component persist the user's report selections into a custom object - call it `Report_Subscription__c`. Include the details (API name) of the selected reports on that object, and a status ('New', 'Current', or 'Cancelled'). Make the Org-Wide Default Private.

Create a Connected App and configure it for JWT authentication in Salesforce, in the same fashion as you would for [using SFDX for continuous integration](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_jwt_flow.htm#!).

Add the profiles of each of the users who are to be subscribed to the Connected App as a pre-approved Profile, or assign all of those users to a Permission Set and assign that Permission Set as pre-approved on the Connected App.

Additionally, ensure that some designated administrator or API user is pre-approved on the Connected App. This user will be employed by the scripts to query for users who need subscriptions maintained, and should have View All permission on `Report_Subscription__c` (or a superseding permission like View All Data).

## Off Platform: Subscription Maintenance Scripts

Then, the backend scripts. This involves some off-Salesforce machine.

You can use SFDX to [authenticate with JWT](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_jwt_flow.htm) and get an access token, just like one does in setting up a Continuous Integration pipeline, or you can implement JWT within your own application. You'll need to do this repeatedly - once for each user to be subscribed, and once at the beginning of the process to locate all of the users who are to be subscribed.

The first job of the script is to authenticate to Salesforce as your administrative user. It should query for all `Report_Subscription__c` records and the usernames of their owners.

Then, it must iterate over the users who have these records present.

For each user, it authenticates to Salesforce as them, again using JWT authentication against our Connected App. For each of that user's `Report_Subscription__c` records that is in status 'New', it should call the Reports and Dashboards API to initiate the subscription as the current user, and update the record to 'Current'. For each that is 'Cancelled', it should call the Reports and Dashboards API to cancel the subscription, and delete the `Report_Subscription__c` record.

This script or scripts will need to run on a regular basis, perhaps nightly. You could do this in a number of ways:

 - Run manually on a local machine. 
 - Run in a cron job on your on-prem or cloud server.
 - Run in Heroku.
 - Repurpose a Continuous Integration pipeline on the Git repository where your scripts are kept to execute it on a schedule ([for example, on CircleCI using scheduled workflows](https://circleci.com/docs/2.0/workflows/#scheduling-a-workflow)).

The key here is that you have to be able to protect your secrets - the private key file for your JWT authentication. I'm most familiar and happy with doing that in a CI context, where you encrypt the key file in your repo and make the passphrase available only through environment variables. But the actual architecture can take a number of different forms.

Personally, I would probably do this as two Python/`simple_salesforce` scripts, calling out to SFDX to handle the authentication and grabbing the auth token out of `sfdx force:org:display`, and I'd have my CI solution execute the script every night, because that's the most efficient use of the architecture I already work with. But there's a lot of ways to get it done, and your organization's architecture (and security needs) will dictate a lot of those choices.

As I mentioned, I've not done the full build-out of an app like this, but I've been thinking about it recently and I believe it's fundamentally feasible. (Edit: I've done a small PoC, below). Feedback and critique would be quite welcome.

## Proof of Concept

I've thrown together a tiny demo that this approach is practical.

Get your Connected App setup for JWT and place your server key in a file `server.key`. Put the username of the user you want to subscribe (must be pre-authorized to the Connected App) in `$USERNAME` and your Connected App's Consumer Key in `$CONSUMERKEY`. Then run

    sfdx force:auth:jwt:grant --clientid $CONSUMERKEY
    --jwtkeyfile server.key --username $USERNAME -a reports-test
    export INSTANCE_URL=$(sfdx force:org:display --json -u reports-test | python -c "import json; import sys; print(json.load(sys.stdin)['result']['instanceUrl'])")
    export ACCESS_TOKEN=$(sfdx force:org:display --json -u reports-test | python -c "import json; import sys; print(json.load(sys.stdin)['result']['accessToken'])")

That establishes an authenticated session as `$USERNAME`, even though we do not have that user's credentials or any setup for that user besides preauthorizing their profile on the Connected App.

Then execute this Python script

    python add-subscription.py $REPORTID

where `$REPORTID` is the Salesforce Id of the report you wish to subscribe the user for, and `add-subscription.py` is the following script:

    import simple_salesforce
    import os
    import sys
    
    outbound_json = """
    {
      "active" : true,
      "createdDate" : "",
      "deactivateOnTrigger" : false,
      "id" : "",
      "lastModifiedDate" : "",
      "name" : "New Notification",
      "recordId" : "%s",
      "schedule" : {
        "details" : {
          "time" : 3
        },
        "frequency" : "daily"
      },
      "source" : "lightningReportSubscribe",
      "thresholds" : [ {
        "actions" : [ {
          "configuration" : {
            "recipients" : [ ]
          },
          "type" : "sendEmail"
        } ],
        "conditions" : null,
        "type" : "always"
      } ]
    }"""
    
    # Use an Access Token and Report Id to add a Lightning report subscription for this user
    # such that the report will run as that user.
    
    access_token = os.environ['ACCESS_TOKEN']
    instance_url = os.environ['INSTANCE_URL']
    
    report_id = sys.argv[1]
    
    sf = simple_salesforce.Salesforce(session_id=access_token, instance_url=instance_url)
    
    sf._call_salesforce(
        'POST',
        sf.base_url + 'analytics/notifications',
        data=outbound_json % report_id
    )

(Yes, I know I shouldn't call `simple_salesforce`'s internal API like that! One could also do this with no more than `bash` and `curl`, if desired.)

And then if we log in as that user in the UI, we'll find a shiny new Lightning report subscription established for them. Note that it's set for daily at 0300, as specified in the example JSON.

[![enter image description here][1]][1]

## Limits Notes

The Reports and Dashboards API has relatively low limits. As noted above, your users can only subscribe to 5 reports. If you're actually running the reports yourself via the API, be aware of the [limits involved](https://developer.salesforce.com/docs/atlas.en-us.api_analytics.meta/api_analytics/sforce_analytics_rest_api_limits_limitations.htm) (trimmed here):
The maximum subscription count for a user [is 5](https://developer.salesforce.com/docs/atlas.en-us.api_analytics.meta/api_analytics/sforce_analytics_rest_api_limits_limitations.htm). 
>  - The API can process only reports that contain up to 100 fields selected as columns.
 - **Your org can request up to 500 synchronous report runs per hour.**
 - **The API supports up to 20 synchronous report run requests at a time.**
 - A list of up to 2,000 instances of a report that was run asynchronously can be returned.
 - The API supports up to 200 requests at a time to get results of asynchronous report runs.
 - Your organization can request up to 1,200 asynchronous requests per hour.
Asynchronous report run results are available within a 24-hour rolling period.
 - The API returns up to the first 2,000 report rows. You can narrow results using filters.


  [1]: https://i.stack.imgur.com/OSaPw.png
