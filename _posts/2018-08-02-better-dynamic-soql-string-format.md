---
layout: post
title: Templating with `String.format()` and `String.join()` for Better Dynamic SOQL 
---

Dynamic SOQL with complex queries and filters can easily become an unreadable mess. Consider a query like this one: 

    String query = 'SELECT Id ' +
        'FROM ' + objectName +
        'WHERE ' + contactLookupName + '.Title = :title ' +
        'AND ' + filterField + ' = \'' + filterValue + '\'' +
        'AND ' + startDateField + ' <= ' + dateValue
        'AND ' + endDateField + ' >= ' + dateValue;

    return query;

What it's *trying* to do is query a dynamically-determined child object of Contact for records associated to Contacts with a specific title, and which are valid for a specific date.

This query also exhibits (at least) five common issues with Dynamic SOQL, in no particular order:

 1. Issues with spacing. A missing space after both `objectName` and `filterValue` results in a parse error at runtime, since Dynamic SOQL syntax cannot be checked at compile time. These errors are very easy to make in Dynamic SOQL and aren't always apparent upon inspection.
 2. Hanging Apex variable binds. The `:title` bind will be evaluated against the local scope at the time this query string is passed to `Database.query()`, not at the time of its creation. This makes storing and passing query strings as parameters fraught with risk: either you must construct and use all query strings within a single scope, or you must not use Apex binding (for which see more below), or you must risk evaluating binds in a different scope and introduce a non-compiler-checkable dependency in your code.
 3. SOQL injection vulnerabilities. Presumably user-supplied values (`filterField`, `filterValue`) aren't escaped with `String.escapeSingleQuotes()` to mitigate potential injection attacks.
 4. Incorrect `Date` format conversion. While it's legal and compiles, implicit `Date`-to-`String` conversion by doing `myString + myDate` results in the wrong `Date` format for SOQL and will yield runtime errors. It's critical to convert Dates to SOQL format with `String.valueOf(myDate)`, or to use Apex binding instead (resurfacing issue 2).
 5. Lack of readability and maintainability. The query is hard to read. It's hard to format the code well. It's hard to parse out what is user input and what's not, and to get a sense of what the query's overall structure looks like.

I strongly prefer to use a query template string with Dynamic SOQL, coupled with `String.format()`. This structure helps maintain a clean separation between the static core and dynamic parameters of the query, and can help mitigate issues 1 and 5, while making it easier to see and address issues 2, 3, and 4. Here's what the query above might look like in that form:

    final String queryTemplate;
    
    queryTemplate = 'SELECT Id FROM {0} WHERE {1}.Title = ''{2}'' AND {3} = ''{4}'' AND {5} <= {6} AND {7} >= {6}';

    return Database.query(
        String.format(
            queryTemplate
            new List<String> {
                String.escapeSingleQuotes(objectName),
                String.escapeSingleQuotes(contactLookupField),
                String.escapeSingleQuotes(title),
                String.escapeSingleQuotes(filterField),
                String.escapeSingleQuotes(filterValue),
                String.escapeSingleQuotes(startDateField),
                String.valueOf(dateValue),
                String.escapeSingleQuotes(endDateField)
            }
        )
    );

By making this switch, we increase the readability of the query, clearly showing which elements are dynamic and which are the static core. We eliminate spacing issues. The compile-time type checking on the `List<String>` ensures that we cannot (unless we have additional logic in the parameters to `new List<String>{}`) perform implicit `Date` conversion, or other implicit type conversions. 

While we're still vulnerable to SOQL injection, explicitly listing out our parameters helps make clear which user-controlled values might need to be escaped. (Note that we're escaping every user-supplied parameter here, even those for which a rogue quote doesn't pose an injection risk; this keeps the static analyzer happy!)

`String.format()` isn't a silver bullet: you still have to remain aware of Dynamic SOQL issues.

While this structure encourages us to use string substitution in lieu of Apex binds, we can still use binds and still encounter dangling bind issues. We'd typically want to retain use of binds anywhere a collection is used with `IN`. It's best to keep those binds within a single method, rather than returning a query string that contains bind values, to avoid those dangling references.

It's easier to show that user input is correctly escaped in this format, but it's still not enforced by the compiler. A good static analyzer is needed to locate such issues. 

Lastly, `String.format()` comes with a few interesting wrinkles due to its inheritance of the format used by Java's [`MessageFormat`](https://docs.oracle.com/javase/7/docs/api/java/text/MessageFormat.html). The most obvious consequence of this is the need to repeat all single quotes in the template string, as above (`''{0}''`). See the Java documentation for more nuances involved in this approach.

## Dynamic `SELECT` Clauses

Dynamic SOQL templating doesn't do anything whatsoever for enforcement of field-level security, which we need to handle in our code. Building `SELECT` clauses dynamically is another common source of string errors, too. Luckily, we can build on `String.format()` templating to handle dynamic `SELECT` clauses cleanly by using collections and `String.join()`.

It's tempting to build a dynamic `SELECT` query by just performing string concatenation, but this approach is inelegant and error-prone.

    String query = 'SELECT Id'; // base query
    // Add fields the user picked
    for (String fieldNameOne : userSelectedFields) {
        query = query + ', ' + fieldNameOne;
    }
    // Add other fields from a field set or Custom Metadata
    for (String fieldNameTwo : someObject.fieldSetA) {
        if (!query.contains(fieldNameTwo)) {
            query = query + ', ' + fieldNameTwo;
        }
    }
    // Do more work to build the query...

is about as clean as this approach gets, and I've seen far worse. It's easy to make simple mistakes that cause hard-to-debug syntax errors, field deduplication to avoid `QueryException`s is messy (note that the above example doesn't actually work in all cases!), and it's hard to handle FLS properly.

Much better is to build the `SELECT` clause by constructing a `List<String>` whose contents are the API names of the desired fields. This offers three benefits:

 1. Easy production of a final, valid `SELECT` clause by applying `String.join()`, and templating in with `String.format()`.
 2. Easy deduplication of the list with `List.contains()`.
 3. Easy enforcement of FLS.

One natural pattern for accumulating and querying a list of fields with FLS enforcement goes like this (note that this assumes none of the selected fields include relationship paths):

    List<String> fieldList = new List<String> { 'id', 'name' };
    List<String> userSuppliedFields = getUserSuppliedFields();
    String sobjectName = 'Contact';

    // Build a single deduplicated list of fields.
    for (String s : userSuppliedFields) {
        if (!fieldList.contains(s)) {
            // List.contains() is a case-sensitive comparison.
            fieldList.add(s.toLowerCase());
        }
    }

    // Iterate over fieldList and check accessibility for the object we're using
    Schema.DescribeSObjectResult dsr = Schema.getGlobalDescribe().get(sobjectName).getDescribe();

    for (String f : fieldList) {
        if (!dsr.fields.getMap().get(f).getDescribe().isAccessible()) {
            // We don't have read permission on this field.
            // Perform some validation action - remove this field,
            // display a PageMessage or Toast, etc.

            // Note that we omit an important check - validating
            // that user input is a real field! `NullPointerException` 
            // can result from the above `if` if `getUserSuppliedFields() 
            // doesn't implement this.
        }
    }

    // then query:

    String query = String.format(
        'SELECT {0} FROM {1}',
        new List<String> {
            String.join(fieldList, ', '),
            sobjectName
        }
    );

So there we are: clean, secure, testable Dynamic SOQL with just a touch of the Standard Library.