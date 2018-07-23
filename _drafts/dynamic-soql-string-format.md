It's true that you cannot bind the `FROM` in a SOQL query, but you certainly can build the entire query dynamically. (You switched to SOSL in the second example; was that expediency or objective?) As an illustration:

    return Database.query(
        String.format(
            'SELECT Name FROM {0} WHERE Name LIKE \'\'{1}\'\' ' +
            'ORDER BY LastViewedDate DESC'
            new List<String> {
                custObj,
                '%' + searchKey + '%'
            }
        )
    )
