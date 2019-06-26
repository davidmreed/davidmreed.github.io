---
layout: post
title: Delta Deployments to Salesforce using ForceCLI and Git
---

Even with the advent of the wonderful Salesforce DX, I have projects where I still prefer to use [ForceCLI](https://github.com/ForceCLI/force) to manage deployments. It has a few great virtues:

 1. It is a single binary with no dependencies, which makes it easy and fast to install on a container image. Just do 

      curl https://force-cli.heroku.com/releases/v0.25.0/linux-amd64/force

 2. It uses Metadata API source format natively, a feature only recently released for Salesforce DX.
 3. It makes it very easy to do delta deployments - pushing only those components that changed within a Git commit range to the server. For very large orgs, using deltas massively reduces the time required for deployments and continuous integration workflows.

ForceCLI has a few issues, though. As of v0.25.0, it struggles with unpacked static resources and with path references to entities within an Aura component bundle, like `helper.js`. To patch around those issues, I use a simple shell script to pack static resources and construct a delta deployment list by squashing bundled path resources.

Here's what that looks like. Suppose that a commit range is supplied in `$1` and `$2`, and `$CHECK_ONLY` is either `-c` or blank. Then, we do

    # Compress unpacked static resources
    for file in src/staticresources/* ; do 
    if [[ -d "$file" && ! -L "$file" ]]; then
        cd $file
        zip -r ../`basename "$file"`.resource *
        cd ../../..
    fi
    done

    # Delta deploy items changed between our start and end commit
    # We use `sed` to squash all references to Aura bundle components and static resources to their parent components
    # `sort -u` ensures that we only have a single reference to each component.
    git diff $1 $2 --name-only --diff-filter=ACM | grep "^src" | sed -E 's#^src/aura/([^/]+)/.+#src/aura/\1#' | sed -E 's#^src/staticresources/([^/]+)/.+#src/staticresources/\1\.resource#' | sort -u | force push $CHECK_ONLY -l RunLocalTests -r -f -
