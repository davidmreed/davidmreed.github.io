---
layout: post
title: Salesforce Lifecycle and Tooling: CircleCI and Salesforce DX
---

This is the first in a series looking at setting up a Salesforce project with a full suite of modern software engineering tools and services. 

As a test-bed, I'm using a project called [septaTrains](https://github.com/davidmreed/septaTrains), a transit tracker application for Lightning. The application is built with Lightning components and uses Apex to call various REST APIs to display information about commuter trains in the Philadelphia area.

We're going to walk through, from the very first step, the tooling for this project. Here, we'll focus on setting up Salesforce DX and Git, and establishing a continuous integration architecture, using CircleCI, with testing automation (for Apex) and code coverage metrics using Codecov.io. In later articles, we'll cover tooling on the IDE side, incorporate PMD static analysis, and add the Lightning Testing Service to the mix, so our JavaScript can be covered too.

Command line snippets below assume a Unix-like environment (Linux/Mac OS X), but should work the same in Git Bash on Windows. PowerShell users will need to make more adaptations.

## Set Up Infrastructure

If you're not using a Dev Hub through a paid Salesforce account, create a new Dev Hub [trial account](https://developer.salesforce.com/promotions/orgs/dx-signup). Trial accounts are good for 30 days, after which they're deleted. Our CI setup is largely resilient to dev hub changes, with minor setup needed but no commits.

Create an SFDX project: 

    $ sfdx force:project:create --projectname proj

If your Dev Hub isn't on `login.salesforce.com`, change the parameter "sfdcLoginUrl" in the `sfdx-project.json` configuration file appropriately.

Set up your Git repository:

    $ cd proj
    $ git init .

Create your Git repository on GitHub and copy its URL.

Configure your local repository to communicate with GitHub and commit your project structure:

    $ git remote add origin <GITHUB_URL>
    $ git add * && git commit -m "Initial commit"


## Create Keys

CircleCI will need to talk to the dev hub at commit time to create a new SFDX scratch org and run  tests. To make this possible, we need to set up the infrastructure for authorizing CircleCI with the JWT flow. JWT doesn't involve any user interaction or require storage of user credentials, making it ideal for CI. To use this solution, we need to create a certificate, which will be stored in a Connected App in our dev hub, and an associated private key, which we'll store in encrypted form in our repository and use to authenticate.

Create your key pair and server certificate as described in the [SFDX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_key_and_cert.htm).

    $ openssl genrsa -des3 -passout pass:x -out server.pass.key 2048
    $ openssl rsa -passin pass:x -in server.pass.key -out server.key
    $ rm server.pass.key
    $ openssl req -new -key server.key -out server.csr
    $ openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
    $ rm server.csr

Generate a symmetric encryption key using your password manager of choice and save it securely - you'll need it now, to encrypt the server private key, and later, to decrypt the key during CI.

Move the server private key to an `assets` directory at the project root:

    $ mkdir assets 
    $ mv server.key assets

Encrypt the server private key (replacing `$KEY` with your symmetric encryption key): 

    $ openssl aes-256-cbc -k $KEY -in assets/server.key -out assets/server.key.enc -e

Remove the unencrypted private key:

    $ rm assets/server.key

Commit the `assets` directory and encrypted private key to Git:
    
    $ git add assets
    $ git commit -m "Add encrypted key"`

## Build Connected App

 Set up a Connected App for the JWT authorization flow, as in the [SFDX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_connected_app.htm), by completing the following steps.
 1. In Lightning Setup, access App Manager and select "New Connected App".
 2. Populate an arbitrary application name and your email address.
 3. Choose "Enable OAuth Settings".
 4. Enter `http://localhost:1717/OauthRedirect` as the callback URL
 5. Select "Use Digital Signatures" and upload the `server.crt` file generated in the last step. 
 6. File the `server.crt` file away somewhere safe, especially if you're ever going to set up the CLI flow with a different org. Don't commit it to Git.
 7. Add the "Access and manage your data", "Perform requests on your behalf at any time", and "Provide access to your data via the Web" OAuth scopes.
 8. Save the connected app.
 9. When the page finishes reloading, copy the Consumer Key and stash it securely (don't commit it to Git).
 10. Click "Manage", then "Edit Policies".
 11. Set Permitted Users under "OAuth policies" to "Admin approved users are pre-authorized".
 12. Click "Save", then "Manage Profiles" (or Permission Sets) and add the Profile or Permission Set assigned to the user you'd like to connect CircleCI under. If you're using a Dev Hub trial, add the System Administrator profile.

## Set up CircleCI for Authentication

Finally, CircleCI itself must be configured to talk to the Dev Hub using the private key. We'll do this using CircleCI Environment Variables, which are stored securely and aren't part of the repository. Follow these steps to complete the CircleCI setup:

1. Add your project to CircleCI after authenticating with GitHub (Projects->Add Project).
2. In the Settings for your project in CircleCI, choose Environment Variables.
3. Add the following environment variables:
    - CONSUMERKEY, with the consumer key from your Connected App in Salesforce.
    - KEY, with the value of the key used to encrypt `server.key`.
    - USERNAME, with the user name of your Dev Hub user account.
  Since CircleCI won't allow you to access the values of these variables later, make sure to retain `$KEY` in a secure store, like a password manager.

## Build `config.yml`

We'll need a `config.yml` file to tell CircleCI how to build and test our project. We're using CircleCI 2.0, which is very different in format from CircleCI 1.0.

First, create the CircleCI directory and YAML file:

    $ mkdir .circleci && touch .circleci/config.yml

Next, construct your `config.yml` file. The `config.yml` for this septaTrains is designed as a [template](https://github.com/davidmreed/septaTrains/blob/master/.circleci/config.yml). It is broken down section-by-section below.

Add `.circleci` to Git and push: 

    $ git add .circleci
    $ git commit -m "Add CircleCI integration"
    $ git push
    
The project builds in SFDX on CircleCI, and if all goes well, you get a green checkmark in your commit log in GitHub! 

CircleCI will monitor commits to the repository and immediately test, and provide feedback on, every single commit, using and disposing a fresh scratch org to make sure code can be cleanly deployed and tested at all times. You'll also get code coverage reports in Codecov.io, and each commit will show your progressive changes in coverage. 

Congratulations - you have a fully-fledged CI solution in place!

If you later need to alter the Dev Hub org for the project, simply create a new Dev Hub user account, ensure that a Connected App is in place in that org (you can use the same certificate), and update the `CONSUMERKEY` and `USERNAME` variables on CircleCI. The remainder of the setup can remain constant.

## `config.yml` for CircleCI

This `config.yml` builds each commit in a new scratch org using SFDX and runs all Apex tests, storing test results in CircleCI. After completing the build, the scratch org is thrown away. It also includes optional functionality to monitor code coverage.

### Base Setup

    version: 2
    jobs:
      build:
        docker:
          - image: circleci/node:latest
          
We're using CircleCI 2.0 with the latest Node.js image as our base. This makes it easy to install SFDX from the command line.

### Caching

        steps:
          - checkout
          - restore_cache:
              keys:
                  - sfdx-version-41-local
                 
The caching structure is one area where you may wish to make changes. We're using a constant cache key, `sfdx-version-41-local`, to force CircleCI to retain SFDX version 41. SFDX's `force:apex:test:run` semantics are documented to be changing after version 41, and we don't want our CI flow to suddenly break because Node installed a newer version of SFDX. Your project may demand a more sophisticated caching strategy, particularly if you have other dependencies.

### Installing SFDX

          - run:
              name: Install Salesforce DX
              command: |
                  openssl aes-256-cbc -k $KEY -in assets/server.key.enc -out assets/server.key -d
                  export SFDX_AUTOUPDATE_DISABLE=true
                  export SFDX_USE_GENERIC_UNIX_KEYCHAIN=true
                  export SFDX_DOMAIN_RETRY=300
                  npm install sfdx-cli
                  node_modules/sfdx-cli/bin/run --version
                  node_modules/sfdx-cli/bin/run plugins --core
          - save_cache:
              key: sfdx-version-41-local
              paths: 
                  - node_modules
                  
First, we'll install SFDX using `npm` and decrypt our server key using the `$KEY` environment variable we previously set up. As noted above, if this is the first run, we'll save our SFDX infrastructure under the constant cache key.
               
### Perform CI 
                
          - run: 
              name: Create Scratch Org
              command: |
                  node_modules/sfdx-cli/bin/run force:auth:jwt:grant --clientid $CONSUMERKEY --jwtkeyfile assets/server.key --username $USERNAME --setdefaultdevhubusername -a DevHub
                  node_modules/sfdx-cli/bin/run force:org:create -v DevHub -s -f config/project-scratch-def.json -a scratch
                  
We need a fresh scratch org for this CI run. We authenticate to our dev hub using the JWT flow, which requires no user interaction and uses the key file we just decrypted. We ask for a new scratch org, which is created synchronously.
                  
          - run:
              name: Remove Server Key
              when: always
              command: |
                  rm assets/server.key
                  
We'll use a `when: always` step to make sure that the unencrypted version of the server key is deleted immediately after use.
                  
          - run: 
              name: Push Source
              command: |
                 node_modules/sfdx-cli/bin/run force:source:push -u scratch
          - run:
              name: Run Apex Tests
              command: |
                  mkdir ~/apex_tests
                  node_modules/sfdx-cli/bin/run force:apex:test:run -u scratch -c -r human -d ~/apex_tests
                  
Next, we push our source up to the new scratch org and ask SFDX to run all Apex tests. With a `-r` argument, this command is run synchronously in SFDX v.41, which we've preferentially cached. When a new version of SFDX is released, we'll update `config.yml` accordingly. Despite `-r human`, SFDX will provide a JUnit XML file that CircleCI can interpret, as well as a human-compatible readout. Note that we're also supplying `-c` to request a code coverage JSON file.

### Teardown, Store Artifacts and Test Results
                  
          - run: 
              name: Push to Codecov.io
              command: |
                  cp ~/apex_tests/test-result-codecoverage.json .
                  bash <(curl -s https://codecov.io/bash)
                  
We're using [codecov.io](https://codecov.io) for code coverage metrics because they support the SFDX code coverage file format. This section is easily separable if you don't want to track code coverage.
                  
          - run: 
              name: Clean Up
              when: always
              command: |
                  node_modules/sfdx-cli/bin/run force:org:delete -u scratch -p
                  rm ~/apex_tests/*.txt ~/apex_tests/test-result-7*.json
                  
To avoid having scratch orgs pile up and hit our limit, we immediately enqueue our new scratch org for deletion. We also remove the redundant test files created by SFDX (it stores several similar or identical formats) so that our `store_test_results` step doesn't grab them and double-count our test metrics in CircleCI.
                  
          - store_artifacts:
              path: ~/apex_tests
          - store_test_results:
              path: ~/apex_tests
              
Finally, we store the test results and code coverage artifacts using CircleCI's native support.

## Resources

- The [sfdx-travisci](https://github.com/forcedotcom/sfdx-travisci) example repository from Salesforce.
- ["Wire It All Together"](https://trailhead.salesforce.com/modules/sfdx_travis_ci/units/sfdx_travis_ci_wire_it) from the Continuous Integration with Salesforce DX Trailhead module.
- [CircleCI 2.0 Docs](https://circleci.com/docs/2.0/)
- The following sections from the [SFDX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_intro.htm):
   - [Authorize an Org Using the JWT-Based Flow](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_jwt_flow.htm)
   - [Create a Connected App](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_connected_app.htm)
   - [Create a Private Key and Self-Signed Digital Certificate](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_key_and_cert.htm)