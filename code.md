---
layout: page
title: Code
---

My full-time position is in developing and applying the [Salesforce.org toolchain](https://github.com/SFDO-Tooling), particularly [CumulusCI](https://github.com/SFDO-Tooling/CumulusCI).

You can find all of my personal projects at [GitHub](https://github.com/davidmreed?tab=repositories) and [GitLab](https://gitlab.com/davidmreed). I work primarily on the Salesforce platform and in Python.

## [Amaxa](https://gitlab.com/davidmreed/amaxa)

Amaxa is a multi-object ETL tool/data loader for Salesforce. It's designed to extract and load whole networks of records, like a selected set of Accounts with all of their Contacts, Opportunities, Contact Roles, and Campaigns, in a single repeatable operation while preserving the relationships between those records.

Core use cases for Amaxa include sandbox seeding, data migration, and retrieving connected data sets.

## Unmaintained Projects

### [septaTrains](https://github.com/davidmreed/septaTrains)

[![CircleCI](https://circleci.com/gh/davidmreed/septaTrains.svg?style=svg)](https://circleci.com/gh/davidmreed/septaTrains)
[![codecov](https://codecov.io/gh/davidmreed/septaTrains/branch/master/graph/badge.svg)](https://codecov.io/gh/davidmreed/septaTrains)

A SEPTA Regional Rail train tracker for Salesforce Lighting. Why? Because it's an interesting test-bed for new development tools and techniques, including Salesforce DX, Codecov.io Apex support, and continuous integration with SFDX and CircleCI.

### [DMRNoteAttachmentImporter](https://github.com/davidmreed/DMRNoteAttachmentImporter)

[![CircleCI](https://circleci.com/gh/davidmreed/DMRNoteAttachmentImporter.svg?style=svg)](https://circleci.com/gh/davidmreed/DMRNoteAttachmentImporter)
[![codecov](https://codecov.io/gh/davidmreed/DMRNoteAttachmentImporter/branch/master/graph/badge.svg)](https://codecov.io/gh/davidmreed/DMRNoteAttachmentImporter)

Note and attachment importing support for Salesforce - add rich-text notes with the Data Loader. This package abstracts the complex content preparation required and helps to avoid unpredictable `ContentNote` exceptions.

### [dedupe_trees.py](https://github.com/davidmreed/dedupe_trees.py)

[![CircleCI](https://circleci.com/gh/davidmreed/dedupe_trees.py.svg?style=svg)](https://circleci.com/gh/davidmreed/dedupe_trees.py)
[![codecov](https://codecov.io/gh/davidmreed/dedupe_trees.py/branch/master/graph/badge.svg)](https://codecov.io/gh/davidmreed/dedupe_trees.py)

A tool to compare and deduplicate divergent file trees, especially with different organization or hierarchy levels.

### [fix15](https://github.com/davidmreed/fix15)

A Python tool to convert 15-character Salesforce Ids in a CSV file to 18-character Ids.

### [SFDX CircleCI Examples](https://github.com/davidmreed/circleci-sfdx-examples)

A repository of examples showing how to apply CircleCI and Salesforce DX in a continuous integration workflow for Salesforce, including demonstrations of testing against multiple org shapes, using PMD static analysis, and applying the Lightning Testing Service.

### [SFDX with `simple_salesforce`](https://github.com/davidmreed/sfdx-simplesalesforce)

A demonstration of using Salesforce DX to execute integration tests against Salesforce scratch orgs for a Salesforce API client written in Python.

### [Trapeza](https://github.com/davidmreed/trapeza) and [Trapeza-Import](https://github.com/davidmreed/trapeza-import)

These Python-based tools are designed to facilitate the manipulation and combination of record data in tabular form - in particular, CSV files containing database output. Trapeza-Import is designed to streamline the importing of large amounts of records into a database by matching duplicates in user-configurable ways. Originally designed for use with Blackbaud Raiser's Edge.

### [iMan](https://github.com/davidmreed/iman)

A graphical manpage viewer for Mac OS X.