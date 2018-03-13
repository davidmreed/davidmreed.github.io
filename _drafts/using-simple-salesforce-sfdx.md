---
layout: post
title: Scripting Salesforce with Python and `simple_salesforce` 
---

Salesforce developers and administrators often encounter situations in org maintenance, sandbox setup, or data manipulation that involve lengthy series of data operations: extracting data, transforming it in some fashion, and reinserting it; moving data trees from one organization to another; seeding sandboxes with generated data or custom settings; or or performing other setup. These workflows can be a major time drain, but at the same time don't merit adoption of a full-scale ETL solution.

Enter `simple_salesforce`. `simple_salesforce` is a Python module providing low-level access to the Salesforce API. It makes it easy to develop one-off or reuseable Python scripts to manage these often ad-hoc data manipulation needs.

    #!/usr/bin/env python

    from simple_salesforce import Salesforce 
    import csv

    sf = Salesforce(username='YOUR_USER_NAME', password='YOUR_PASSWORD', security_token = 'YOUR_SECURITY_TOKEN')

    writer = csv.DictWriter(f = sys.stdout, fieldnames=['Id', 'FirstName', 'LastName'])
    writer.writeheader()
    for record in sf.query_all('SELECT Id, FirstName, LastName FROM Contact').get('records'):
        writer.writerow({key: record[key] for key in writer.fieldnames})

This tiny script (just 10 lines of Python) runs a query and saves the results to a .CSV file. While this could be done with Workbench or the Data Loader, what makes the Python/`simple_salesforce` approach so valuable is what we can do to extend this script.