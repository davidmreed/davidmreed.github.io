---
layout: post
title: Integrating Static Analysis with PMD in the Salesforce Development Lifecycle
---

Static analysis is a powerful complement to unit tests, helping to identify bugs early in the development lifecycle and root out dangerous code practices - even before the source is compiled. [PMD](http://pmd.github.io/) is a multi-language static analysis tool that includes support for Apex and, to a limited extent, Visualforce.

Static analysis can be added to the development lifecycle at multiple levels. PMD can be run within the IDE as code is being written to identify flaws before they're even compiled or pushed to the server. It can also run across an entire codebase as part of a continuous integration solution, to head off problems merging in, enforce code style and good practices, identify locations of technical debt, and surface subtle bugs that may not be flagged by unit tests.

## Static Analysis in the IDE: Visual Studio Code

Visual Studio Code offers the [Apex PMD](https://marketplace.visualstudio.com/items?itemName=chuckjonas.apex-pmd) by Charlie Jonas that adds PMD support for Apex. To set up the extension, first [download PMD](https://pmd.github.io/#downloads) and decompress the ZIP file into a location where the application can live (you'll need to provide its path to the extension).

Then, install the extension in Visual Studio Code. Open your User Preferences and search for `apexPMD`. Beside the setting `apexPMD.pmdPath`, click the pencil icon and copy it your settings. Add the path to the extracted PMD directory here. (For example, `/Users/dreed/Projects/pmd-bin-6.0.1` - you want the full path to the top-level PMD directory).

You can also configure the `apexPMD.runOnFileOpen` and `apexPMD.runOnFileSave` settings to determine when static analysis is performed. Apex PMD feeds all static analysis warnings into the Visual Studio Code Problems view.

## Static Analysis in the IDE: Eclipse

PMD provides an official plugin for Eclipse, which works well with Apex. Install the plugin within Eclipse by following the instructions [supplied by PMD](https://pmd.github.io/latest/pmd_userdocs_tools.html#eclipse)

## Static Analysis with Continuous Integration: Code Climate

Code Climate does not officially support Apex. Fortunately, an open source engine called [ApexMetrics](https://github.com/rsoesemann/codeclimate-apexmetrics) can be activated within Code Climate to apply PMD static analysis in an automated, CI-compatible fashion.

## Static Analysis with Continuous Integration: CircleCI

## Developing Rule Sets

