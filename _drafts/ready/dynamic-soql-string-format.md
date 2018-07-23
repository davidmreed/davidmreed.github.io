Dynamic SOQL with complex queries and filters can easily become an unreadable mess. Consider a query like this one: 

    String query = 'SELECT Id ' +
        'FROM ' + objectName +
        'WHERE ' + contactLookupName + '.Title = :title ' +
        'AND ' + filterField + ' = ' + filterValue + 
        'AND ' + startDateField + ' <= ' + dateValue
        'AND ' + endDateField + ' >= ' + dateValue;

    return query;

What it's *trying* to do is dynamically query a child object of Contact that's determined at runtime, for records associated to Contacts with a specific title, and which are valid for a specific date. 

This query also exhibits (at least) five common issues with Dynamic SOQL, in no particular order:

 1. Issues with spacing. A missing space after both `objectName` and `filterValue` results in a parse error at runtime (since Dynamic SOQL syntax cannot be checked at compile time). These errors are very easy to make in Dynamic SOQL and aren't always apparent upon inspection.
 2. Hanging Apex variable binds. The `:title` bind will be evaluated against the local namespace at the time this query string is passed to `Database.query()`, not at the time of its creation. This makes storing and passing query strings as parameters fraught with risk: either you must construct and use all query strings within a single namespace, or you must not use Apex binding (for which see more below), or you must risk evaluating binds in a different namespace and introduce a non-compiler-checkable dependency in your code.
 3. SOQL injection vulnerabilities. Presumably user-supplied values (`filterField`, `filterValue`) aren't escaped with `String.escapeSingleQuotes()` to mitigate potential injection attacks.
 4. Incorrect `Date` format conversion. While it's legal and compiles, implicit `Date`-to-`String` conversion by doing `myString + myDate` results in the wrong `Date` format for SOQL and will yield runtime errors. It's critical to convert Dates to SOQL format with `String.valueOf(myDate)`, or to use Apex binding instead (resurfacing issue 2).
 5. Lack of readability and maintainability. The query is hard to read. It's hard to format the code well. It's hard to parse out what is user input and what's not, and to get a sense of what the query as a whole looks like.

I strongly prefer to use a query template string with Dynamic SOQL, coupled with `String.format()`. This structure helps maintain a clean separation between the static core and dynamic parameters of the query, and can help mitigate issues 1 and 5, while making it easier to see and address issues 2, 3, and 4. Here's what the query above might look like in that form:

    final String queryTemplate;
    
    queryTemplate = 'SELECT Id FROM {0} WHERE {1}.Title = {2} AND {3} = ''{4}'' AND {5} <= {6} AND {7} >= {6}';

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

While this structure encourages us to use string substitution in lieu of Apex binds, we can still use binds and still encounter dangling bind issues. We'd typically want to retain use of binds anywhere a collection is used with `IN`. It's best to keep those binds within a single method, rather than returning a query string that contains bind values.

By making this switch, we greatly increase the readability of the query, clearly showing which elements are dynamic and which are the static core. We completely eliminate spacing issues. The compile-time type checking on the `List<String>` ensures that we cannot (unless we have additional logic in the parameters to `new List<String>{}`) perform implicit `Date` conversion. While we're still vulnerable to SOQL injection, explicitly listing out our parameters helps make clear which user-controlled values might need to be escaped.

The biggest pitfall of using `String.format()` is that the parameter it accepts is in the format used by Java's [`MessageFormat`](https://docs.oracle.com/javase/7/docs/api/java/text/MessageFormat.html). The most obvious consequence of this is the need to repeat all single quotes in the template string, as above (`''{0}''`). See the Java documentation for more nuances.