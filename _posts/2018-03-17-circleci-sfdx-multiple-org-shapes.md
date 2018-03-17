---
layout: post
title: "Salesforce Lifecycle and Tooling: Testing on Multiple Org Types with Salesforce DX and CircleCI Workflows"
---

Let's suppose you're running a successful continuous integration program, using [Salesforce DX and CircleCI]({{ site.baseurl }}{% post_url 2018-02-02-salesforce-dx-circleci %}) or another continuous integration provider. Your automated testing is in place, and working well. But the code you're building has to work in a number of different environments. You might be an ISV, an open-source project, or an organization with multiple Salesforce instances and a shared codebase, and you need to make sure your tests pass in both a standard Enterprise edition and a Person Accounts instance, or in Multi-Currency, or a Professional edition, or any number of other combinations of Salesforce editions and features.
n
Salesforce DX and CircleCI make it very easy to automate running tests against these different Salesforce environments, and to do so in efficient, parallel, isolated testing streams. The process is built in three steps:

 1. Define organization types and features in Salesforce DX [Scratch Org Definition Files](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file_config_values.htm) in JSON format.
     - Optionally, define additional Metadata API or package installation steps to complete preparation of a specific org for testing.
 1. Define jobs in CircleCI configuration file, either by scripting each environment's unique setup individually or by referencing a common build sequence in the context of each org type.
 1. Define a workflow in CircleCI configuration file that runs these jobs in parallel.
 
This article assumes that you've followed [Salesforce Lifecycle and Tooling: CircleCI and Salesforce DX]({{ site.baseurl }}{% post_url 2018-02-02-salesforce-dx-circleci %}) and are using a fairly similar `config.yml`. However, the principles are transferable to other continuous integration environments and build sequences.
 
## Defining Organization Types and Features
 
 Salesforce DX scratch org definitions don't need to be complex, and are defined in JSON. This is a simple example that adds a feature (Sites) to the default configuration: 
 
     {
        "orgName": "David Reed",
        "edition": "Enterprise",
        "features": ["Sites"],
        "orgPreferences" : {
            "enabled": ["S1DesktopEnabled"]
        }
    }

The feature set that is accessible through the org definition file is still somewhat in flux. New features are being added, and some important facets are still not available. The best references for what *is* available are the [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file_config_values.htm) and the [Salesforce DX](https://success.salesforce.com/_ui/core/chatter/groups/GroupProfilePage?g=0F93A000000HTp1) group in the Trailblazer Community.

Org definition files live in the `config` directory in a DX project. When you create a scratch org, you provide a definition file with the `-f` switch; you're free to add multiple definition files to your repository.

Note that we're not here discussing the Org Shape feature, which is currently in pilot. Once Org Shape becomes publicly available, more capabilities will become available for defining and creating types of environment.

## Define Jobs in CircleCI

Each organization definition we want to test against is represented as a `job` entry in the CircleCI `config.yml`.

    version: 2
    jobs:
      - build-enterprise
      - build-developer
      ...
      
We can define an arbitrary number of these jobs. 

If we define jobs by copying and pasting the [core SFDX build job]({{ site.baseurl }}{% post_url 2018-02-02-salesforce-dx-circleci %}), our `config.yml` can become unwieldy and difficult to maintain. If there's a lot of setup work that significantly differs between the org definitions, it might be necessary nonetheless. 

However, if the job definitions vary by little more than the name of the scratch org definition file, we can take advantage of YAML's aliasing feature to template our core build instructions into each job, while using environment variables to define the differences between them.

Here's what it looks like. (The complete `config.yml` file is [available on GitHub](https://github.com/davidmreed/septaTrains/blob/master/.circleci/config.yml)).

    job-definition: &jobdef
        docker:
            - image: circleci/node:latest
        steps:
            ...

`&jobdef` defines an alias, a name to which we can refer to include the following material, which we've factored out from the core `config.yml` developed previously. To that core build sequence, we make just one change, in the "Create Scratch Org" step:

        - run: 
            name: Create Scratch Org
            command: |
                node_modules/sfdx-cli/bin/run force:auth:jwt:grant --clientid $CONSUMERKEY --jwtkeyfile assets/server.key --username $USERNAME --setdefaultdevhubusername -a DevHub
                echo "Creating scratch org with definition $SCRATCH_DEF"
                node_modules/sfdx-cli/bin/run force:org:create -v DevHub -s -f "config/$SCRATCH_DEF" -a scratch

Note that we're using a new environment variable, `$SCRATCH_DEF`, to store the name of the definition file we want to use. We'll take advantage of that when we template this core mechanic into the individual jobs that define builds for each type of org.

Below this alias definition, at the top level of `config.yml`, we'll start our `jobs` list:

    version: 2
    jobs:
      build-enterprise:
         <<: *jobdef
         environment:
            SCRATCH_DEF: project-scratch-def.json
      build-developer: 
         <<: *jobdef
         environment:
            SCRATCH_DEF: developer.json

Here, we define two jobs, one per scratch org. Each one *includes* the entire core build sequence `&jobdef`, including all of the build steps we've defined. Within each job, we assign a value to the environment variable `$SCRATCH_DEF`, which the build will use to create its unique scratch org.

Each of these jobs will run in a separate, isolated container, and each will use its own scratch org. We'll get independent test results for each org definition, ensuring that our code's behavior is correct in each org separately from the others.

This form can be extended even if your different org definitions require more configuration than is possible through the definition file. For example, each org might require installation of a different managed package with (for example) `sfdx force:package:install -i $PACKAGE_ID`. Or you might need to perform a different Metadata API deployment with `sfdx force:mdapi:deploy -d "$MD_API_SRC_DIR" -w 10`. Provided the build processes are *structurally* similar, templating and environment variables can help express them concisely and make the build easy to maintain.

There's always the option, though, of copying and modifying our core build sequence into any individual job or set of jobs, making as many modifications as necessary. CircleCI will run them all the same, whichever route we take.

## Complete the Process with CircleCI Workflow

The final step is to create a `workflow` entry in `config.yml`. The workflow ties together the different build jobs and expresses any dependencies between them. Lacking dependencies, the jobs will run in parallel, using as many containers as you have available.

    workflows:
      version: 2
      test_and_static:
        jobs:
          - build-enterprise
          - build-developer
          - static-analysis
          
Here, we define a three-job workflow - one each for the two org definitions against which we want to test, and a third job for our PMD static analysis (see [Integrating Static Analysis with PMD in the Salesforce Development Lifecycle]({{ site.baseurl }}{% post_url 2018-02-08-static-analysis-pmd-salesforce %})). When we push to Git, CircleCI will initiate these three jobs in parallel. Each will succeed or fail individually, and you'll get status indicators in GitHub for each job.

![GitHub Results]({{ site.baseurl }}/public/multi-org-shape/github-results.png)

The workflow as a whole shows success or failure aggregated from its component jobs, and you can rerun the entire workflow or individual failed jobs as needed.

So there we have it: our code, tested instantly and efficiently against as many different Salesforce orgs as we need - subject, of course, to your organization's scratch org limits! 
