---
layout: post
title: `fix15` - A New Tool to Support Data Loads and ETLs
---

I've released a new Python tool, [`fix15`](https://github.com/davidmreed/fix15). This tool aims to simplify Salesforce data-load and ETL workflows by seamlessly converting 15-character Id values to their corresponding 18-character Ids in CSV data.

There are existing online Id converters, but I wanted a solution I could use from the command line as I prepped and manipulated files for data load, without copying and pasting or using complex, very slow array formulas. With `fix15`, just do

> `fix15 -c Id -c AccountId -i test.csv -o done.csv`

to convert the columns "Id" and "AccountId" in the file `test.csv` and write the result to `done.csv`.

The tool is tiny, tested, and MIT-licensed. It has no dependencies outside the Python standard library and doesn't require access to Salesforce (or, for that matter, a network connection).