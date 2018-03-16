---
layout: post
title: "Salesforce DX Continuous Integration: Testing on Multiple Org Types"
---

Let's suppose you're running a successful continuous integration program, using Salesforce DX and CircleCI or another continuous integration provider. Your automated testing is in place, and working well. But the code you're building has to work in a number of different environments. You might be an ISV, an open-source project, or an organization with multiple Salesforce instances and a shared codebase, and you need to make sure your tests pass in both a standard Enterprise edition and a Person Accounts instance, or in Multi-Currency, or a Professional edition, or any number of other combinations of Salesforce editions and features.

Salesforce DX and CircleCI make it very easy to automate running tests against these different Salesforce environments, and to do so in efficient, parallel, isolated testing streams. The process is built in three steps:
 1. Define organization types and features in Salesforce DX [Scratch Org Definition Files](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file_config_values.htm) in JSON format.
   - Optionally, define additional Metadata API or package installation steps to complete preparation of a specific org for testing.
 1. Define jobs in CircleCI configuration file, either by scripting each environment's unique setup individually or by referencing a common build sequence in the context of each org type.
 1. Define a workflow in CircleCI configuration file that runs these jobs in parallel.
 
 ## Defining Organization Types and Features
 
 Salesforce DX scratch org definitions don't need to be complex. This is a simple example that adds a feature (Sites) to the standard 
 
     {
        "orgName": "David Reed",
        "edition": "Enterprise",
        "features": ["Sites"],
        "orgPreferences" : {
            "enabled": ["S1DesktopEnabled"]
        }
    }

The feature set that is accessible through the org definition file is still somewhat in flux. New features are being added, and some important facets are still not available. In some cases you may be able to perform Metadata API deploys to help rectify these shortfalls. The best references for what's available are the [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file_config_values.htm) and the [Salesforce DX](https://success.salesforce.com/_ui/core/chatter/groups/GroupProfilePage?g=0F93A000000HTp1) group in the Trailblazer Community.

Org definition files live in the `config` directory in a DX project. When you create a scratch org, you provide a definition file with the `-f` switch; you're free to add multiple definition files to your repository.

## Define Jobs in CircleCI

## Complete the Process with CircleCI Workflow
