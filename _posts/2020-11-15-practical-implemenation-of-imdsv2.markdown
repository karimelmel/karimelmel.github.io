---
layout: post
title:  "It's time to implement Instance Metadata Service V2"
date:   2020-11-15
---

Were getting close to a year ago since AWS launched the Instance Metadata Service Version 2 (IMDSv2) to protect the metadata service and more importantly the [instance profile credentials from four known vulnerabilities](https://aws.amazon.com/blogs/security/defense-in-depth-open-firewalls-reverse-proxies-ssrf-vulnerabilities-ec2-instance-metadata-service/) and more than a year since CapitalOne suffered their related breach. Since then I've yet to come across an organization that actively enforces IMDSv2 and I believe this will remain the case until it becomes the default.

For my own part, I decided it is time to start implementing the IMDSv2, afterall it has been out for a year now, we all know exactly what code needs to be updated so it was just a matter of getting it done.

### Discovering and enforcing usage of IMDSv2
AWS has a great guide on the steps towards implementation of this and [you can even monitor usage of IMDS version 1 in Cloudwatch](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html#instance-metadata-transition-to-version-2). The guidance also covers how to change existing instances using AWS CLI, but with  teams that are in control of their own IAM policies, telling them to implement it is unlikely to ensure 100% coverage.

Because of that enforcing it through an SCP may seem natural [like Scott suggests how to achieve here](https://summitroute.com/blog/2020/03/25/aws_scp_best_practices/#require-the-use-of-imdsv2). If you ever tried that in an organization you probably had to turn it off relatively quick no matter how well you communicated the change, because developers will still struggle to implement this and I'll tell you why later in this post.

### Implementing IMDSv2 with Cloudformation
Now when you have decided to enforce usage you need your developers to start implementing it. If your using the AWS CLI, deploying an EC2 with IMDSv2 is pretty straight forward and covered in the AWS documentation.

If you are using CloudFormation however, it is a bit more tricky and I'll cover the steps here. 
First you need to deploy a [AWS::EC2LaunchTemplate](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-launchtemplate-launchtemplatedata.html) resource, it can be either be a part of the EC2 stack or deployed separately.

The template for this is as following:
```
Resources:  
  IMDSv2LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
        LaunchTemplateName: IMDSV2
        LaunchTemplateData:
          MetadataOptions:
            HttpEndpoint: enabled
            HttpPutResponseHopLimit: 1
            HttpTokens: required
```
{: .language-yaml}

Deploying it separately might be your best option, since depending on how you use CloudFormation to deploy EC2 instances, you may only have to deploy it once and reference it across multiple EC2 resources.

To then deploy an instance using [AWS::EC2::Instance](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html) where you later reference the Launch Template by the value of LaunchTemplateName that we set to be "IMDSV2" in the stack above where we deployed our Launch Template. 

The CloudFormation stack for the EC2 would then look like this:
```
Resources:
  EC2InstanceExample:
    Type: AWS::EC2::Instance
    Properties:
        InstanceType: t2.micro
        LaunchTemplate: 
          LaunchTemplateName: IMDSV2
          Version: 1
        ImageId: ami-0947d2ba12ee1ff75
```
{: .language-yaml}

### What about Auto-Scaling Groups and ECS?
So when I started testing for an Auto-Scaling Groups it confirmed my suspicion that this cannot be widely adopted by organizations, [the documentation for CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-autoscaling-autoscalinggroup-launchtemplate.html) was wrong, so I submitted a PR to correct this [https://github.com/awsdocs/aws-cloudformation-user-guide/pull/857](https://github.com/awsdocs/aws-cloudformation-user-guide/pull/857). 
Once the PR is merged the documentation should be correct. 

Your template will then look like the below, that also includes deployment of ECS:

```
Resources:
AutoScalingExample:
  Type: AWS::AutoScaling::LaunchConfiguration
  Properties:
    MetadataOptions:
      HttpEndpoint: enabled
      HttpPutResponseHopLimit: 1
      HttpTokens: required
```
{: .language-yaml}


That's it for now. Let me know on Twitter if I missed any service or if you have better ways of enforcing it.