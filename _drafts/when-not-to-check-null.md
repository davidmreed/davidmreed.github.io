---
layout: post
title: Defensive Error Checking Gone Wrong
---

Starting out as a developer is difficult. 

The Apex language isn't as simple as it seems, and it doesn't come with some of the niceties that modern general-purpose programming languages have. The mind-share penetration of recent _massive_ improvements in developer tooling still isn't as high as it could be - and for junior developers, those tools are just another of a thousand things to learn!

So too often, junior developers finds themselves confronting what feels like a barrage of errors whose causes they don't intuit, without the tools or conceptual frameworks or available work time to adequately dissect them, and react by writing defensive Apex: code designed less to behave as expected under the myriad of conditions encountered in production and more to make the compiler and the runtime be quiet.

### Checking for Impossible Nulls and Nonexistent Errors

Here's a common example of error checking that doesn't need to be there when working with DML operations:

    List<Account> accounts = new List<Account>();
    
    for (Account a : [SELECT Id, ... FROM Account WHERE ...]) {
        // Do something with `a`, populate `accounts`.
    }
    
    if (accounts != null && accounts.size() > 0) {
        update accounts;
    }
    
This check - `if (accounts != null && accounts.size() > 0)` - is useless. **Collection constructors don't return `null`**. There's no possibility that `accounts` is `null` in this code, because it has been initialized to a value that is guaranteed not to be `null`. Apex isn't C; allocating an object will never yield a `null`.

Further, there's no need to check whether there are any Accounts in the list before performing DML. DML on an empty collection is a no-op: nothing happens, and no limits are consumed.

### Overzealous Exception Handling and Hiding
