---
layout: post
title: Don't Check for Errors that Can't Happen
---

I wrote last week about not writing bad exception handlers in Apex. Today, I'd like to touch on another area of error handling 

Here's a common example of error checking that doesn't need to be there when working with DML operations:

```apex
List<Account> accounts = new List<Account>();

for (Account a : [SELECT Id, ... FROM Account WHERE ...]) {
    // Do something with `a`, populate `accounts`.
}

if (accounts != null && accounts.size() > 0) {
    update accounts;
}
```
    
This check - `if (accounts != null && accounts.size() > 0)` - is not particularly harmful, but it is useless. It decreases the readability of the code and increases its cyclomatic complexity. There's no possibility that `accounts` is `null` in this code, because it has been initialized to a value that is guaranteed not to be `null`. Apex isn't C; allocating an object will never yield a `null` value.

Further, there's no need to check whether there are any Accounts in the list before performing DML. DML on an empty collection is a no-op: nothing happens, and no limits are consumed.

So these three lines, and every three lines resembling them, can be replaced with a simple

```apex
update accounts;
```

This change clarifies the logic, reduces the complexity of the code, and removes the *impression* of error checking being performed when in fact no error is possible.

The same type of redundant error checking is often seen around iteration and other collection operations:

```apex
if (!accounts.isEmpty()) {
    for (Account a : accounts) {
        // Do work with `a`
    }
}
```

The outer `if` is unnecessary, because iterating over an empty (*not `null`, but empty*) collection is also a no-op. The list body never executes.

Again, removing the excess `if` checking, which accomplishes nothing, streamlines the code and increases readability.
