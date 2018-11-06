---
layout: post
title: Running Reports as Selected Users with JWT Oauth and the Reports and Dashboards API
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

While the Reporting and Analytics API doesn't support setting the context user for a subscription, it does take action as the user as whom we authenticate to the API. And that we **can** control.

While an admin can Login As a user to create a one-off subscription, we're more interested here in industrial-strength solutions that can support thousands of users. So we're going to build a script to create subscriptions by talking to the Reports and Dashboards API, using the Javascript Web Token (JWT) Oauth authentication mechanism. Why? Because the JWT flow is our only route to seamlessly authenticating as *any* (admin-approved) user, with no manual intervention or setup required on a per-user basis.

## Setup: Connected Apps and Certificates

Setting up the JWT flow involves building a Connected App in Salesforce, under which our scripts will authenticate. JWT is secured using a certificate and associated public key/private key pair - Salesforce holds the public key, our script holds the private key. 

This is the same mechanism used for authentication in many Continuous Integration solutions. I'm not going to rehash all of the details here, because they're well-covered elsewhere. You can follow Salesforce's steps in [using SFDX for continuous integration](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_jwt_flow.htm#!), or read through my own article about [setting up CircleCI with Salesforce DX](https://www.ktema.org/2018/02/02/salesforce-dx-circleci/). 

When you're finished building the Connected App, add the Profiles of each of the users who are to be subscribed to reports to the Connected App as a pre-approved Profile, or assign all of those users a Permission Set and assign that Permission Set as pre-approved on the Connected App. This ensures that we can authenticate to the API as those users without any intervention.

## Building the Scripts

We're going to stick to sketching out a solution here that can be adapted to many different business problems, as we discussed earlier. For simplicity, we'll use Salesforce DX to handle the JWT authentication, even though we're not using SFDX for development here. Because it's my preferred scripting workflow, I'll be using Python with `simple_salesforce`, but you could just as easily achieve this in Ruby, Java, JavaScript, or even just bash and curl.

The main job of our script is to login as a user and create a report subscription for them. We might build this towards a specific business process by adding scaffolding to, for example, query a custom object out of Salesforce to define *which* reports should be subscribed automatically for which users, but we'll leave that elaboration to a later date. Once we've got that core functionality achieved, we can wrap it in the logic we need for specific applications.

Let's put the key field (private key) from our JWT setup in a file called `server.key`. Put the username of the user we want to subscribe (who must be pre-authorized to the Connected App) in the environment variable `$USERNAME` and the Connected App's Consumer Key in `$CONSUMERKEY`. 

Then we can get an Access Token to make an API call into Salesforce, letting SFDX do the heavy lifting:

    sfdx force:auth:jwt:grant --clientid $CONSUMERKEY
    --jwtkeyfile server.key --username $USERNAME -a reports-test
    export INSTANCE_URL=$(sfdx force:org:display --json -u reports-test | python -c "import json; import sys; print(json.load(sys.stdin)['result']['instanceUrl'])")
    export ACCESS_TOKEN=$(sfdx force:org:display --json -u reports-test | python -c "import json; import sys; print(json.load(sys.stdin)['result']['accessToken'])")

Now we've established an authenticated session as `$USERNAME`, even though we do not have that user's credentials or any setup for that user besides preauthorizing their profile on the Connected App, and we have the values we need (the Access Token and Instance URL) stored in our environment.

Now we'll switch over to Python. A simple script grabs those environment variables and uses `simple_salesforce` to make an API call to generate the report subscription.

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

Execute the script

    python add-subscription.py $REPORTID

where `$REPORTID` is the Salesforce Id of the report you wish to subscribe the user for, and then if we log in as that user in the UI, we'll find a shiny new Lightning report subscription established for them. 

FIXME: image here

Note that it's set for daily at 0300, as specified in the example JSON.

## Building the Business Process

### Bulk One-Off Subscriptions

### On-Platform Subscription App

This script or scripts will need to run on a regular basis, perhaps nightly. You could do this in a number of ways:

 - Run manually on a local machine. 
 - Run in a cron job on your on-prem or cloud server.
 - Run in Heroku.
 - Repurpose a Continuous Integration pipeline on the Git repository where your scripts are kept to execute it on a schedule ([for example, on CircleCI using scheduled workflows](https://circleci.com/docs/2.0/workflows/#scheduling-a-workflow)).

The key here is that you have to be able to protect your secrets - the private key file for your JWT authentication. I'm most familiar and happy with doing that in a CI context, where you encrypt the key file in your repo and make the passphrase available only through environment variables. But the actual architecture can take a number of different forms.

Personally, I would probably do this as two Python/`simple_salesforce` scripts, calling out to SFDX to handle the authentication and grabbing the auth token out of `sfdx force:org:display`, and I'd have my CI solution execute the script every night, because that's the most efficient use of the architecture I already work with. But there's a lot of ways to get it done, and your organization's architecture (and security needs) will dictate a lot of those choices.
