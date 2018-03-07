---
layout: post
title: Deduplicating File Trees with Python 
---

I have three computers and two phones, and I use Dropbox to sync photos between all of my devices. But from time to time I've broken the Dropbox sync for some reason, or changed my partition table, or set up a machine not to sync, or I'd start (and perhaps not finish) an organization project. The end result was a solid half-dozen different versions of my photo archive, which had diverged from one another not only in content and in editing status but in organization too.

I needed a way to merge these archives together without losing edited versions of photos or undoing the album organization that was present in some, but not all, of the directories. To solve that issue, I wrote [`dedupe_trees`](https://github.com/davidmreed/dedupe_trees.py). 

This Python tool walks a set of directories and deduplicates them in a configurable way, letting you prefer newer or older file versions, versions from one tree over another, versions located deeper or shallower in your folder hierarchy, and drop versions that appear to be simple copies. It can remove duplicates, label them for your handling, or move them to a separate file tree. Most importantly, you can apply a sequence of duplicate resolvers - like "first remove all obvious copies, then prefer the version sorted deeper in the tree, then choose the most recently modified version". 

`dedupe_trees` works on Linux and Mac OS X (provided Python 3 is installed) and is under the MIT License. And yes, I did test it on my photo archive (*after* all the unit tests passed). It worked.
