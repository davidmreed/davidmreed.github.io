---
layout: post
title: Integrating Static Analysis with PMD in the Salesforce Development Lifecycle
---

This is the second in a series looking at setting up a Salesforce project with a full suite of modern software engineering tools and services. (See the first for more on [setting up CI with Salesforce DX](http://www.ktema.org/2018/02/02/salesforce-dx-circleci/))

Static analysis is a powerful complement to unit tests, helping to identify bugs early in the development lifecycle and root out dangerous code practices - even before the source is compiled. [PMD](http://pmd.github.io/) is a multi-language static analysis tool that includes support for [Apex](https://pmd.github.io/pmd-6.0.1/pmd_rules_apex.html) and, to a limited extent, [Visualforce](https://pmd.github.io/pmd-6.0.1/pmd_rules_vf.html).

Static analysis can be added to the development lifecycle at multiple levels. PMD can be run within the IDE as code is being written to identify flaws before they're even compiled or pushed to the server. It can also run across an entire codebase as part of a continuous integration solution, to head off problems merging in, enforce code style and good practices, identify locations of technical debt, and surface subtle bugs that may not be flagged by unit tests.

PMD works by applying *rules*, which define specific issues to look for in a code base. Rules are supplied as part of the application. (Contributors can write new rules in Java). *Rule sets*, however, can be defined by end user on an application-by-application basis. Part of the work of incorporating static analysis into the development lifecycle is defining the most effective rule set for each engagement point (IDE, continuous integration). We'll discuss rule set definition at the end; you may end up with several different rule sets to be applied at different stages of the development lifecycle.

Be aware that PMD is not a truly "plug-and-play" tool, and while its support for Apex is good, it may not be at the level of source review that you expect from other platforms. PMD is particularly useful for highlighting some of the most dangerous practices, like SOQL and DML in loops, and for enforcing fairly basic stylistic rules. (Note that it can't enforce fine-grained brace or whitespace styles). Below, we'll look at several different options for integration points in the Salesforce development lifecycle.

## Static Analysis in the IDE: Visual Studio Code

Visual Studio Code can use the [Apex PMD](https://marketplace.visualstudio.com/items?itemName=chuckjonas.apex-pmd) extension by Charlie Jonas. To set up the extension, first [download PMD](https://pmd.github.io/#downloads) and decompress the ZIP file into a location where the application can live (you'll need to provide its path to the extension). PMD requires Java, which is likely already installed as part of the Salesforce DX setup process.

Then, install the extension in Visual Studio Code. Open your User Preferences and search for `apexPMD`. Beside the setting `apexPMD.pmdPath`, click the pencil icon and copy it your settings. Add the path to the extracted PMD directory here. (For example, `/Users/dreed/Projects/pmd-bin-6.0.1` - you want the full path to the top-level PMD directory).

You can also configure the `apexPMD.runOnFileOpen` and `apexPMD.runOnFileSave` settings to determine when static analysis is performed. Apex PMD feeds all static analysis warnings into the Visual Studio Code Problems view, and they're also surfaced as green underlines in the source code view. (Mouse over the affected area for warning details). You can invoke the static analyzer explicitly from the command palette with `Apex Static Analysis: On File` or `Apex Static Analysis: On Workspace` to get a codebase-wide analysis.

![Apex PMD results in Visual Studio Code]({{ site.baseurl }}/public/apex-pmd/apex-pmd-results.png)

Apex PMD comes with a comprehensive rule set. However, you'll likely want to define your own, to silence warnings that aren't relevant or change the warning level for specific rules. See the section below on Developing Rule Sets to create a rule set. Once you've created your own rule set, enable it by setting `apexPMD.useDefaultRuleset` to `false`, and populating the path to your rule set in `apexPMD.rulesetPath`.

## Static Analysis in the IDE: Eclipse

PMD provides an official plugin for Eclipse, which works well with Apex. Install the plugin within Eclipse by following the instructions [supplied by PMD](https://pmd.github.io/latest/pmd_userdocs_tools.html#eclipse). Note that the plugin includes PMD itself, so you do not also need to install the application separately.

Once installed, PMD is controlled in the Project Properties window. A custom rule set can be designated here, but the Eclipse plugin also allows for the individual activation and deactivation of specific rules, helping to reduce the need for rule set development.

![Eclipse PMD Configuration]({{ site.baseurl }}/public/apex-pmd/pmd-project-config.png)

Unlike the Visual Studio Code extension, Eclipse's PMD plugin runs on demand. To run the static analyzer, right-click a source code file, a directory, or a project, and choose PMD->Check Code. Static analyzer results are shown in one or more special panes, accessible via Window->Show View->Other->PMD. The Violations Outline pane shows a simple list of flagged areas in the code; they're also shown with colored indicators in the source code view's gutter area. Color coding is counterintuitive; green translates to "urgent" violations, and files in the explorer are badged with a muddy color blend of their violation lists. Violations can be cleared from the PMD section of the context menu.

![Eclipse PMD Violations]({{ site.baseurl }}/public/apex-pmd/pmd-violations-view.png)

## Static Analysis in the Cloud: Code Climate

Code Climate does not officially support Apex, but an open source engine called [ApexMetrics](https://github.com/rsoesemann/codeclimate-apexmetrics) can be activated to apply PMD static analysis in an automated, CI-compatible fashion.

Code Climate offers a low-setup, automated mechanism for running static analysis and tracking flaws visually. Unfortunately, the integration with ApexMetrics could be a bit smoother. Apex issues are surfaced only in an "Other Issues" section and don't count towards maintainability metrics. Rule application is controlled by rule set files or by entries in the `.codeconfig.yml` configuration file, or both, but not through the web application.

Code Climate uses a fairly old version of PMD ([5.5.3, dated to January 2017](https://github.com/pmd/pmd/releases/tag/pmd_releases%2F5.5.3)). Many important rules have been added to PMD since that release, and further, the rule set format has been revamped. This means that rule sets used in other PMD contexts (including ApexMetrics' own examples, based on PMD 6.0) won't work in Code Climate. It's necessary to either source a rule set file from an old release of PMD as a starting point for designing your own rule set or to exclusively use `.codeclimate.yml` to manage rule selection. The [full rule set](https://github.com/pmd/pmd/blob/847ea1c0843eec2d35923e71f4bc904c1c4ed601/pmd-apex/src/main/resources/rulesets/apex/ruleset.xml) from PMD 5.5.3 is a good starting point for building a rule set file.

Additional attributes can be included in the `ruleset.xml` file to categorize violations for Code Climate. The stock Apex rule sets from PMD, which are designed for use with ApexMetrics, provides examples of this configuration.

Code Climate requires a `.codeclimate.yml` configuration file. This file should be committed to the repository *before* adding the repository to Code Climate, so that the initial run will take Apex settings into account. An example `.codeclimate.yml` file is available on a [branch](https://github.com/davidmreed/septaTrains/blob/codeclimate/.codeclimate.yml) of septaTrains. It simply disables all built-in checks provided by Code Climate, ignores static resources, and activates the ApexMetrics engine. Note that Code Climate has updated their configuration file format and examples provided by ApexMetrics are no longer valid.

`.codeclimate.yml` can also tailor the ruleset further, if you don't want to generate your own rule set or just prefer to avoid editing the XML. Adding a `checks` map under the `apexmetrics` plugin allows suppression of specific rules by name. For example, 

    plugins:
      apexmetrics:
        enabled: true
        checks:
          StdCyclomaticComplexity:
            enabled: false
            
results in suppression of the `StdCyclomaticComplexity` rule, even if it's present in the XML ruleset file. 

![Code Climate Configuration]({{ site.baseurl }}/public/apex-pmd/code-climate-disable-check.png)

To find a rule name in Code Climate, mouse over a violation entry and then the right-most context button ('Exclude Checks'). Click 'Disable Check' to see instructions for modifying `.codeclimate.yml` to suppress that specific warning. Rule names are also included in the `<rule>` entries of a rule set file, as the final element of the path that specifies the rule. (See below for details).

## Static Analysis with Continuous Integration: CircleCI

PMD can be run directly within CircleCI. Plugging the static analyzer directly into the build can be useful if, for example, the build should fail based upon a rule violation, like performing SOQL in a loop. Since CircleCI has no built-in functionality for tracking PMD's defect reports, however, codebase-level static analysis is better delegated to a service like Code Climate or to an IDE plugin unless the (1) the existing codebase is clean for the rule set being used and (2) any new results for the chosen rule set should flag the build as a failure.

Since static analysis can run in parallel with the Salesforce DX build/test operation, this is a great fit for CircleCI 2.0's Workflows. We can define a PMD job that runs independently of Salesforce DX with a new entry under `jobs` in our `config.yml` (starting from the same `config.yml` we used in [setting up CI](http://www.ktema.org/2018/02/02/salesforce-dx-circleci/)), and adding an entry under `workflows`.

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

![GitHub Build Status]({{ site.baseurl }}/public/apex-pmd/github-build-status.png)

## Developing Rule Sets

PMD rule sets are XML files that define which of the built-in rules should be run on source code, and the severity assigned to violations of those rules. Because rule set files comprise semi-cryptic references to PMD's internal hierarchy of rules in a verbose XML format, it's easiest to start with a broad preexisting rule set and trim it down to suit your needs.

A great place to start is the global [rule set](https://github.com/pmd/pmd/blob/master/pmd-apex/src/main/resources/rulesets/apex/ruleset.xml) provided with PMD.

Within the rule set file, each rule is activated by an XML `<rule>` element. Note that the `ref` attribute is a reference to PMD's internal rule categorization, terminating in the rule's name.

    <rule ref="category/apex/performance.xml/AvoidSoqlInLoops" message="Avoid Soql queries inside loops">
        <priority>3</priority>
        <properties>
            <!-- relevant for Code Climate output only -->
            <property name="cc_categories" value="Performance" />
            <property name="cc_remediation_points_multiplier" value="150" />
            <property name="cc_block_highlighting" value="false" />
        </properties>
    </rule>

Rules can be deactivated by simply removing the entire `<rule>` element, or by commenting it out. If necessary, the priority and Code Climate properties can also be changed. Note however that in the CircleCI setup described here, any violation (regardless of priority) marks the build as a failure; a command line option to PMD (`-minimumpriority`) can filter on this basis.

Depending on where you choose to integrate PMD in your development lifecycle, you may have zero, one, or several rule sets. Eclipse and Code Climate don't need them (but can use them); Apex PMD and the CircleCI workflow do need rule sets.

## Recommendations

PMD is a useful tool for maintaining visibility into Apex code quality. It's not a panacea: there are many stylistic and functional issues it won't catch, and it can flag a lot of false positives if you enable its entire rule set. Despite those caveats, it has a place in the development lifecycle as an early warning of risky code practices, and an enforcer of a basic code style.

PMD is perhaps best when applied with a very broad ruleset at the IDE level and a very narrow ruleset at the CI level. Optionally, a cloud service integrating PMD for Apex might provide a broader codebase overview, although Code Climate does have significant limitations. 

I am excited about the promise of a newer tool, Clayton.io, which offers Salesforce-specific static analysis over and above what PMD can do. Clayton is under active development and I understand improvements are forthcoming.