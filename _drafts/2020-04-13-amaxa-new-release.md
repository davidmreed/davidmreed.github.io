---
layout: post
title: "Amaxa: New Release, Easier Installation, and More"
---

Amaxa, my open source, multi-object data loader for Salesforce, is moving forward with a new release. Amaxa 0.9.6 brings new features, bug fixes, and more user-friendly installation on Mac, Windows, and Linux. Just one more major feature before version 1.0!

Here's a summary of what's new in Amaxa 0.9.5 and 0.9.6.

- Prebuilt, single-binary executables are now published for Linux, Mac OS, and Windows 10 (using PyInstaller).
- Amaxa supports setting Bulk API options, including the batch size.
- Amaxa offers more data transform options, and supports plugin modules to extend transformation capabilities.
- Amaxa can use Salesforce DX authentication.
- Environment variables can be referenced in credential files.
- Amaxa now supports a `--check-only` flag to validate a potential operation run.
- Extensive refactoring of the codebase took place to support future enhancements.
- Significantly expanded documentation is now on [ReadTheDocs](https://amaxa.readthedocs.io/).
- The primary repository has moved to [GitHub](https://github.com/davidmreed/amaxa).
- Amaxa is built using Poetry, and is tested on Python 3.6, 3.7, and 3.8 on Mac, Windows, and Linux.

New features require the use of `version: 2` in operation and credential files.

My gratitude for continued feedback, bug reports, design suggestions, and helpful queries goes out to Amaxa's users, who helped to shape this release.

[Install](https://amaxa.readthedocs.io/en/latest/install.html) Amaxa 0.9.6 by downloading an executable for your operating system, or via Python's `pip` package manager.