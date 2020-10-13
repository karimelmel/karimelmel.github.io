---
layout: post
title:  "AWS Detective Account CSV file"
date:   2020-09-25
---

As most should be familiar by now, you can setup [Delegated Admin for both GuardDuty and Access Analyzer](https://summitroute.com/blog/2020/05/04/delegated_admin_with_guardduty_and_access_analyzer/). This feature haven't yet been made available for Detective but Amazon have provided scripts that allows setting up Detective in all accounts with a cross-account role. The scripts be found here [https://github.com/aws-samples/amazon-detective-multiaccount-scripts](https://github.com/aws-samples/amazon-detective-multiaccount-scripts). 

The only thing that was stopping me from automating this for all new accounts is that the CSV list had to be generated, which can be exported from the GUI in GuardDuty or programmatically through the Master account. Below example displays the export capability for GuardDuty master:
![](/image/gdexport.JPG)

I've been searching for a way to do this programmatically, I eventually learned after [ranting on Twitter](https://twitter.com/fvant/status/1313736138499272706) that if GuardDuty is configured through Delegated Admin, it can be retrieved through the [Organizations SDK(https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/organizations.html#Organizations.Client.list_accounts)] but not through the GuardDuty SDK as documented. In the event where GuardDuty is setup with invitiations, it can be retrieved using the GuardDuty SDK as documented.
![](/image/gdexport2.JPG)

With this ability to export the Organizations members, I can now programmatically retrieve a list of accounts and store them in a CSV file without having to access the GUI or have someone export it from the Organization master account by using the code embedded below. Note that this can only run on an account with Delegated Admin or Organizaton Master:

```
# Exports Organizations members as CSV to be used as input for Amazon Detective
import boto3
import json
import csv
import os

client = boto3.client('organizations')

def create_accounts_csv():
    response = client.list_accounts()
    for i in range(len(response)):
        with open('accounts.csv', mode='w', newline='') as accounts_file:
            for i in range(len(response)):
                writer = csv.writer(accounts_file, dialect='excel', delimiter='"',lineterminator='')
                writer.writerow([(response['Accounts'][i]['Email']+',')])
                writer.writerow([(response['Accounts'][i]['Id']+'\n')])

create_accounts_csv()
```
{: .language-python}

### Why would you do this

Until the AWS Detective team releases functionality that allows delegated admin, this may simplify the workflow for setting up new accounts if you have an environment with large account inflation, you can follow the steps and then run this on a schedule without having to provide an updated CSV file manually.

## Additional thoughts

A Delegated Admin for GuardDuty can be used to perform reconnaissance on other accounts in the environment and there may be potential trusts towards the GuardDuty account if remediation is hosted within that account, if they have a weak Trust Relationship policy you may be able to assume it as any role.