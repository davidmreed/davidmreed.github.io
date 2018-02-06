---
layout: post
title: Integrating Static Analysis with PMD in the Salesforce Development Lifecycle
---

Static analysis is a powerful complement to unit tests, helping to identify bugs early in the development lifecycle and root out dangerous code practices - even before the source is compiled. [PMD](http://pmd.github.io/) is a multi-language static analysis tool that includes support for [Apex](https://pmd.github.io/pmd-6.0.1/pmd_rules_apex.html) and, to a limited extent, [Visualforce](https://pmd.github.io/pmd-6.0.1/pmd_rules_vf.html).

Static analysis can be added to the development lifecycle at multiple levels. PMD can be run within the IDE as code is being written to identify flaws before they're even compiled or pushed to the server. It can also run across an entire codebase as part of a continuous integration solution, to head off problems merging in, enforce code style and good practices, identify locations of technical debt, and surface subtle bugs that may not be flagged by unit tests.

PMD works by applying *rules*, which define specific issues to look for in a code base. Rules are supplied as part of the application. (Contributors can write new rules in Java). *Rule sets*, however, can be defined by end user on an application-by-application basis. Part of the work of incorporating static analysis into the development lifecycle is defining the most effective rule set for each engagement point (IDE, continuous integration). We'll discuss rule set definition at the end; you may end up with several different rule sets to be applied at different stages of the development lifecycle.

Be aware that PMD is significantly less "plug-and-play" than other current-generation Salesforce tools. It's a valuable asset, but it does take some iteration and research to set up and use effectively. Below, we'll look at several different options for integration points in the Salesforce development lifecycle.

## Static Analysis in the IDE: Visual Studio Code

Visual Studio Code offers the [Apex PMD](https://marketplace.visualstudio.com/items?itemName=chuckjonas.apex-pmd) extension by Charlie Jonas. To set up the extension, first [download PMD](https://pmd.github.io/#downloads) and decompress the ZIP file into a location where the application can live (you'll need to provide its path to the extension). PMD requires Java, which is likely already installed as part of the Salesforce DX setup process.

Then, install the extension in Visual Studio Code. Open your User Preferences and search for `apexPMD`. Beside the setting `apexPMD.pmdPath`, click the pencil icon and copy it your settings. Add the path to the extracted PMD directory here. (For example, `/Users/dreed/Projects/pmd-bin-6.0.1` - you want the full path to the top-level PMD directory).

You can also configure the `apexPMD.runOnFileOpen` and `apexPMD.runOnFileSave` settings to determine when static analysis is performed. Apex PMD feeds all static analysis warnings into the Visual Studio Code Problems view, and they're also surfaced as green underlines in the source code view. You can invoke the static analyzer explicitly from the command palette with `Apex Static Analysis: On File` or `Apex Static Analysis: On Workspace` to get a codebase-wide analysis.

Apex PMD comes with a comprehensive rule set. However, you'll likely want to define your own, to silence warnings that aren't relevant or change the warning level for specific rules. See the section below on Developing Rule Sets to create a rule set. Once you've created your own rule set, enable it by setting `apexPMD.useDefaultRuleset` to `false`, and populating the path to your rule set in `apexPMD.rulesetPath`.

## Static Analysis in the IDE: Eclipse

PMD provides an official plugin for Eclipse, which works well with Apex. Install the plugin within Eclipse by following the instructions [supplied by PMD](https://pmd.github.io/latest/pmd_userdocs_tools.html#eclipse)

Once installed, PMD is controlled in the Project Properties window. A custom rule set can be designated here, but the Eclipse plugin also allows for the individual activation and deactivation of specific rules, helping to reduce the need for rule set development.

Unlike the Visual Studio Code extension, Eclipse's PMD plugin runs on-demand only. To run the static analyzer, right-click a source code file, a directory, or a project, and choose PMD->Check Code. Static analyzer results are shown in one or more special panes, accessible via Window->Show View->Other->PMD. The Violations Outline pane shows a simple list of flagged areas in the code; they're also shown with colored indicators in the source code view's gutter area.

## Static Analysis in the Cloud: Code Climate

Code Climate does not officially support Apex. An open source engine called [ApexMetrics](https://github.com/rsoesemann/codeclimate-apexmetrics) can be activated within Code Climate to apply PMD static analysis in an automated, CI-compatible fashion.

Code Climate does provide a low-setup, automated mechanism for running static analysis and tracking flaws. Unfortunately, the service is geared towards its own internal engines (which don't support Apex), and its integration with ApexMetrics leaves something to be desired in both user interface and functionality. Apex issues will be surfaced only in an "Other Issues" section and don't count towards maintainability metrics.

Additionally, Code Climate uses a fairly old version of PMD ([5.5.3, dated to January 2017](https://github.com/pmd/pmd/releases/tag/pmd_releases%2F5.5.3)), which is more than a year old. In additional to significant new development on Apex rules, PMD has undergone a revamp of its rule set format since 5.5.3. This means that rule sets used in other PMD contexts (including ApexMetrics' own examples) won't work in Code Climate, and it's necessary to source a rule set file from an old release of PMD as a starting point for designing your own rule set.

Code Climate requires a `.codeclimate.yml` configuration file. This file should be committed to the repository *before* adding the repository to Code Climate, so that the initial run will take Apex settings into account. An example `.codeclimate.yml` file is available on a [branch](https://github.com/davidmreed/septaTrains/blob/codeclimate/.codeclimate.yml) of septaTrains. It simply disables all built-in checks provided by Code Climate and activates the ApexMetrics engine.

## Static Analysis in the Cloud: Clayton.io

Clayton.io does not advertise itself as a PMD-based service, but its code results strongly suggest this is the case. Unfortunately, Clayton in my testing proved to be fairly unreliable, although it does offer a nice graphical rule set editor. Its open source offering is also limited to 25,000 lines of code and metadata reviewed per month, a limit that's fairly easy to hit.

## Static Analysis with Continuous Integration: CircleCI

PMD can be run directly within CircleCI. Plugging the static analyzer directly into the build can be useful if, for example, the build should fail based upon a rule violation, like performing SOQL in a loop. Since CircleCI has no built-in functionality for tracking PMD's defect reports,however, codebase-level static analysis is better delegated to a service like Code Climate unless the (1) the existing codebase is clean for the rule set being used and (2) any new results for the chosen rule set should flag the build as a failure.

Since static analysis can run in parallel with the Salesforce DX build/test operation, this is a great fit for CircleCI 2.0's Workflows. We can define a PMD job that runs independently of Salesforce DX with a new entry under `jobs` in our `config.yml` (starting from the same `config.yml` we used in [setting up CI](http://www.ktema.org/2018/02/02/salesforce-dx-circleci/))

    jobs:
      static-analysis:
        docker:
          - image: circleci/openjdk:latest
        steps:
          - checkout
          - restore_cache:
              keys:
                - pmd-v6.0.1
          - run:
              name: Install PMD
              command: |
                  if [ ! -d pmd-bin-6.0.1 ]; then
                      curl -L "https://github.com/pmd/pmd/releases/download/pmd_releases/6.0.1/pmd-bin-6.0.1.zip" -o pmd-bin-6.0.1.zip
                      unzip pmd-bin-6.0.1.zip
                      rm pmd-bin-6.0.1.zip
                  fi
          - save_cache:
              key: pmd-v6.0.1
              paths:
                  - pmd-bin-6.0.1
          - run:
              name: Run Static Analysis
              command: |
                  pmd-bin-6.0.1/bin/run.sh pmd -d . -R $RULESET -f text -l apex -r static-analysis.txt
          - store_artifacts:
              path: static-analysis.txt

All we need to do, then, is commit an XML rule set file to our repository and put its name in the CircleCI `$RULESET` environment variable. A simple workflow definition at the end of `config.yml` ensures that the static analysis runs alongside the existing build process:

    workflows:
      version: 2
      test_and_static:
        jobs:
          - build
          - static-analysis

A failure in either job - build and test with Salesforce DX, or static analysis with PMD - then marks our build as failed in CircleCI and in GitHub.

## Developing Rule Sets

PMD rule sets are XML files that define which of the built-in rules should be run on source code, and the severity assigned to violations of those rules. Because rule set files comprise semi-cryptic references to PMD's internal hierarchy of rules in a verbose XML format, it's easiest to start with a broad preexisting rule set and trim it down to suit your needs.

## Recommendations

Unfortunately, the two cloud services integrating PMD for Apex of which I am aware have significant limitations and issues. Pending improvements and updates to such a service, I recommend integrating PMD static analysis at the IDE level, ideally using Apex PMD with Visual Studio Code. This solution can be used both for day-to-day development support and for overall codebase review. Secondarily, I recommend adding PMD with a limited rule set focused on high-risk issues to the continuous integration flow.
