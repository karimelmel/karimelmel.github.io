---
layout: post
title:  "Enabling tags in Instance Metadata"
date:   2022-01-25
---
## Background
As you may be aware of, AWS has launched the ability to read instance tags from the metadata endpoint, meaning it is possible for an instance to read its tags directly without calling the API. Until now this has also been possible by using an Instance Profile with permissions to run ec2:DescribeTags, then first retrieving the Instance Id from the endpoint metadata service followed by using AWS Cli, PowerShell etc. from the client to DescribeTags.

Now that it can be read directly from the Metadata endpoint, this workaround is no longer needed. As always, there is one caveat with new AWS features, it is rarely (if ever?) enabled 'by default'.

## Enabling tags using Python SDK
The tags can be enabled for all instances in an account using the Python SDK or Cli. A simple example on enabling it with the Python SDK is embedded below:

```
import boto3

client  = boto3.client('ec2')

def get_instances():
    instances = client.describe_instances(
        Filters=[
            {
                'Name': 'instance-state-name',
                'Values': [
                    'running',
                    'stopped',
                ]
            },
        ],
    )
    instance_list=[]
    for r in instances['Reservations']:
        for i in r['Instances']:
            instance_list.append(i['InstanceId'])
    return instance_list



def enable_metadata_tags():
    instance_ids = get_instances()
    for i in instance_ids:
        client.modify_instance_metadata_options(
            InstanceId=i,
            InstanceMetadataTags='enabled'
            )

if __name__ == "__main__":
    enable_metadata_tags()

```
{: .language-python}

## What about new instances?
For new instances, you will have to modify the Instance Metadata Options to opt-in for all your templates, [similar to my post on enabling Instance Metadata Service V2](https://blog.karims.cloud/2020/11/15/practical-implemenation-of-imdsv2.html) Or you can slightly modify the script to run in a Lambda and hook it up to EventBridge, you'll then want it to trigger on the following pattern in EventBridge:

```

{
  "source": ["aws.ec2"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["ec2.amazonaws.com"],
    "eventName": ["RunInstances"]
  }
}
```
{: .language-json}

For the Lambda to run it should have the following policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ModifyMetadataOptions",
            "Effect": "Allow",
            "Action": [
                "ec2:ModifyInstanceMetadataOptions",
                "ec2:DescribeInstances",
            ],
            "Resource": "*"
        }
    ]
}
```
{: .language-json}

And the code below will do the absolutely minimum to run this when a new instance is launched (no logging etc.):

```
import boto3
client  = boto3.client('ec2')

def enable_metadata_tags(event, context):
    detail = event['detail']
    ids = []
    eventname = detail['eventName']
    if eventname == 'RunInstances':
        items = detail['responseElements']['instancesSet']['items']
    for item in items:
        ids.append(item['instanceId'])
        for i in ids:
            client.modify_instance_metadata_options(
                InstanceId=i,
                InstanceMetadataTags='enabled'
                )

```
{: .language-python}

