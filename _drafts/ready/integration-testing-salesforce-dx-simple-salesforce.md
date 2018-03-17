---
layout: post
title: Integration Testing Off-Platform Code with Salesforce DX and `simple_salesforce`
---

I am a huge fan of the lightweight, easy-to-use [`simple_salesforce`](https://github.com/simple-salesforce/simple-salesforce) module for Python. I use `simple_salesforce` constantly, for everything from one-off data analysis scripts to sandbox setup automation to full-scale ETL solutions. As some of those solutions grow more complex or durable, I start to feel the need to built serious tests for them.

For software like this, of course, Python unit tests tell a relatively small part of the story. It's critical to actually connect the tool to a Salesforce instance, kick it off, and make sure the results look right - an end-to-end integration test.

This kind of test isn't repeatable in sandboxes or in a developer edition; "real" orgs get polluted, and it's easy to introduce silent dependencies. That's why Salesforce DX scratch orgs are a great solution for testing *off-platform* code just as much as Apex: we build a scratch org with a predictable shape and feature set, perhaps seed some standardized data, connect our integration tool, run our tests, and toss the scratch org.

With off-platform code there's a couple of extra steps needed to get access to the scratch org, and we have two good options depending on how the integration typically connects to Salesforce. A demonstration of both techniques is available on [my GitHub](https://github.com/davidmreed/sfdx-simplesalesforce) in a CircleCI context.

## Access Token

The easiest route is if the integrated tool can use an access token and instance URL to connect to Salesforce. (This is supported by `simple_salesforce`). Once we've created our scratch org, a couple of SFDX and Python one-liners can extract these values and pass them to the tool under test:

    export ACCESS_TOKEN=$(sfdx force:user:display --json | python -c "import json; import sys; print(json.load(sys.stdin)['result']['accessToken'])")
    export INSTANCE_URL=$(sfdx force:user:display --json | python -c "import json; import sys; print(json.load(sys.stdin)['result']['instanceUrl'])")
    python example-simple-salesforce.py -a "$ACCESS_TOKEN" -i "$INSTANCE_URL"

Some tools may use the environment variables, while others will need the access values on the command line. However, the tool acquires these values, it can establish a connection to the scratch org with `simple_salesforce`:

    connection = simple_salesforce.Salesforce(instance_url=instance_url, session_id=access_token)

## Username and Password

Scratch org access via username and password is a bit trickier. Salesforce DX scratch orgs don't start with passwords, although we can easily create them. More importantly, though, it's not possible ([and likely never will be possible](https://success.salesforce.com/_ui/core/chatter/groups/GroupProfilePage?g=0F93A000000HTp1SAG&fId=0D53A00003EtYb9SAF) - Success Community login required) to set or obtain a scratch org user's security token from the command line.

Fortunately, we *can* use SFDX to access the Metadata API. And with the Metadata API, we can deploy a small `Security.settings` entity that establishes all IP addresses as a trusted network, allowing us to bypass verification codes and the security token requirement.

(Is this a great idea, security-wise? Not especially. But we're only implementing it on an ephemeral scratch org that will be recycled in a matter of minutes. *Don't* do this to your production, sandbox, developer edition, or long-lived scratch org).

    sfdx force:mdapi:deploy -d md-src -w 5

    sfdx force:user:password:generate > /dev/null
    export PASSWORD=$(sfdx force:user:display --json | python -c "import json; import sys; print(json.load(sys.stdin)['result']['password'])")
    export SF_USERNAME=$(sfdx force:user:display --json | python -c "import json; import sys; print(json.load(sys.stdin)['result']['username'])")
    python example-simple-salesforce.py -p "$PASSWORD" -u "$SF_USERNAME" -t "" -s

The Metadata API deployment package is available on [GitHub](https://github.com/davidmreed/sfdx-simplesalesforce). We deploy the package first (with a `-w` wait of up to 5 minutes for the deployment to complete), then ask SFDX to generate a new password for the user. We redirect `stdout` to `/dev/null` because `sfdx force:user:password:generate` prints a human-readable message containing the password to `stdout`, and we don't need that in our CI logs (although, again, we are throwing away this scratch org).

Finally, we once again apply Python one-liners to parse the JSON output of `sfdx force:user:display` and extract the username and password we need. We pass, in whatever format is most appropriate, an empty string to our tool where a security token would normally go, and that tool then authenticates to Salesforce:

    connection = simple_salesforce.Salesforce(username=username,
                                              password=password,
                                              security_token=token,
                                              sandbox=sandbox)

The example tool in GitHub uses `argparse` to accept these values on the command line.

## Performing Tests

Once we have access to SFDX, we'll run one or more scenarios with our integrated tool. Wrapper scripts can use the same approach to gain access to the scratch org and check the results, making appropriate assertions along the way:

    # ... having run a script that creates Accounts

    connection = simple_salesforce.Salesforce(instance_url=instance_url, session_id=access_token)
    results = connection.query('SELECT Id, Name, AnnualRevenue FROM Account')

    assert len(results.get('records')) == 12
    assert 'Test Account 1' in [r['Name'] for r in results.get('records)]
    # ... and so on. We have full access to review and evaluate the state of the scratch org post-testing.