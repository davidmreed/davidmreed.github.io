---
layout: post
title: Use `INDEX()/MATCH()`, Not `VLOOKUP()`, with Salesforce Data Loader 
---

I recently completed a major org-to-org migration on Salesforce and I did not use `VLOOKUP()` once. I want to encourage you to add `INDEX()/MATCH()` to your toolbox for using the Salesforce Data Loader.

`VLOOKUP()` is old and clunky and dangerous. It can't index to the right, so you have to put your ID values in the left hand column then count column indices out to the right to get your data back. Since it takes a cell range rather than a column, it's easy to break the formula if you fill it down across a column (...and forget the make your references absolute). It's error-prone and overcomplicated.

The `INDEX()/MATCH()` construct is vastly better for Data Loader preparation. It's easy to set up and difficult to break. Here's what it looks like.

Given some incoming data with Contact email addresses but no Salesforce Ids, alongside a mapping file of email addresses and corresponding Ids, we can set up `INDEX()/MATCH()` like this:

![Mapping Data]({{ "/public/index-match/index-match-mapping-data.png" | absolute_url }})
![INDEX/MATCH Formula Setup]({{ "/public/index-match/index-match-formula.png" | absolute_url }})

`INDEX()` takes the value from a column (its first argument) at a specific row (the second argument). Hence, for the first argument we provide the column in which we've stored the Salesforce Ids we want to look up. For the second, the row, we use `MATCH()`. `MATCH()` takes its first argument (here, the email address) and returns the row where that value is found in the column identified by the second argument.

Because we use column references rather than rectangular ranges, you can fill the formula down a column without breaking references, even without making them absolute. The construct can index to the right, as it does here, and it's resilient to changes in your data sheets. `INDEX()/MATCH()` is the way to go.

![INDEX/MATCH Results]({{ "/public/index-match/index-match-results.png" | absolute_url }})

Of course, nothing in the Excel world can be quite perfect. `INDEX()/MATCH()` still isn't case-safe, so we're stuck with 18-character Salesforce Ids, or must use excruciatingly slow [array formulas](https://exceljet.net/formula/exact-match-lookup-with-index-and-match) to avoid collisions when Excel treats `001000000000aaj` and `001000000000aaJ` as equivalent values. I'm working on a programmatic solution to make fixing this issue quick and easy - more updates to come!