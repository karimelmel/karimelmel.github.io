---
layout: post
title:  "AWS AI Data Sharing Opt-out"
date:   2020-08-09
---

A few months ago it was discovered that AWS had updated their Terms and Conditions for customer data in certain AI services to be shared with them for the purpose of improving their solution. Customers need to selectively opt-out of data sharing if desired, this was uncovered in an article written in Computer Business Review [https://www.cbronline.com/news/aws-user-data](https://www.cbronline.com/news/aws-user-data). The terms and conditions can also be read below:

![](/image/AWSAI.jpg)
*Source: CBR*

There are a few steps required to disable the sharing that is documented by AWS:
[https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_ai-opt-out_create.html](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_ai-opt-out_create.html)

I decided to create a script using the AWS SDK for Python (boto3) that enables this. My main motivation for creating this script is that I am in the process learning Python and also want to ensure opt-out in my organizations.

<b> To run the script, the following IAM policy is at least required: </b>
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "organizations:EnablePolicyType",
                "organizations:AttachPolicy",
                "organizations:DescribePolicy"
            ],
            "Resource": "arn:aws:organizations::*:account/o-*/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "organizations:ListRoots",
                "organizations:DescribeOrganization",
                "organizations:CreatePolicy",
                "organizations:ListPolicies"
            ],
            "Resource": "*"
        }
    ]
}

```
{: .language-json}

<b> The script below will: </b>
<ol>
<li>List the root-id for the organization</li>
<li>Enable the policy type for AI that is required before you can attach an opt-out policy</li>
<li>Create the AI opt-out policy with scope default, which includes all the services</li>
<li>Attach the policy to the organization root, applying to all accounts in the organization.</li>
</ol>

```
import boto3
import botocore.exceptions
import json
import string
import random

# Script to opt-out of AI Sharing with AWS. 
# https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_ai-opt-out.html

# Defines the boto3 client for Organizations
client = boto3.client('organizations')

# Gets the root of the organization, need for 
def list_roots():
    response = client.list_roots()
    return response['Roots'][0]['Id']

# Enables the policy type to allow setting a policy for AI opt-out.
def enable_policy_type():
    rootid=list_roots()
    try:
        response = client.enable_policy_type(
            RootId=rootid,
            PolicyType='AISERVICES_OPT_OUT_POLICY'
        )
    except botocore.exceptions.ClientError as error:
        print('Policy type have already been enabled.. Skipping.')
        return error
    else:
        print('Successfully enabled the policy type.')
        return response


# Defines the policy with default scope to opt-out from sharing information on all services
ai_services_policy = {
    "services": {
        "default": {
            "opt_out_policy": {
                "@@assign": "optOut"
            }
        }
    }
}
ai_services_policy = json.dumps(ai_services_policy)


# Creates the AI Opt-Out policy
def create_policy():
    try:
        uid = ''.join(random.choices(string.ascii_uppercase + string.digits, k=5))
        response = client.create_policy(
            Content=ai_services_policy,
            Description='Organization Policy to opt-out of AI sharing',
            Name='ai-opt-out-'+uid,
            Type='AISERVICES_OPT_OUT_POLICY'
        )
    except botocore.exceptions.ClientError as error:
        print('A policy with duplicate name already exists..')
        return error
    else:
        print('Policy has been created.. needs to be attached before it takes effect.')
        return response ['Policy']['PolicySummary']['Id']


# Attaches the policy that has been created
def attach_policy():
    policyId = create_policy()
    targetId = list_roots()
    try:
        response = client.attach_policy(
            PolicyId=policyId,
            TargetId=targetId
        )
    except botocore.exceptions.ClientError as error:
        print('The policy has already been attached or no policyId was returned.. skipping.')
        raise  error
    else:
        print('AI Opt-Out policy has been successfully attached to the organizations root.')
        return response
        
        
if __name__ == '__main__':
    enable_policy_type()
    attach_policy()
```
{: .language-python}

The script can also be found here: [https://github.com/karimelmel/aws-python-samples/blob/master/ai-opt-out.py](https://github.com/karimelmel/aws-python-samples/blob/master/ai-opt-out.py).

Reach out to me on [Twitter](https://twitter.com/karimmelhaoui) if you have any feedback.