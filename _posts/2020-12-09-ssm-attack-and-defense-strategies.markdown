---
layout: post
title:  "AWS Systems Manager Attack and defense strategies"
date:   2020-12-08
---
## Background
AWS Systems Manager is long known for its ["Run Command"](https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html) allowing an attacker to move from AWS API Access to compromise instances that may be within the network. This post will try to cover multiple aspects of Systems Manager, Documents and how to protect and detect techniques leveraging this. This post will focus on Windows Server.

## Documents
Systems Manager has an ecosystem of documents that can be shared either directly with a specified account or globally for all AWS customers. The documents can be shared from any AWS accounts, this is similar to AMIs and poses the same risk if an untrusted SSM document is ran. There is also the opportunity to be creative with names and one may pretend a document is from a specific vendor, as some vendors provide documents. The only Owner not verified by the AccountId is Amazon.
![](/image/ssmdocuments.png)

### Uploading a malicious document
The format for documents is farily simple and scripts can be provided on multiple lines as PowerShell Code. For the purpose of this demo I've setup [Empire C2](https://github.com/bc-security/empire) with a http listener where I've generated an obfuscated payload that I then upload to an SSM document in account A. 
![](/image/aws_ssm_document.png)

I then modify the permissions on the document to allow public sharing so it appears for all AWS accounts in the world.
![](/image/sharing.png)

### Running malicious documents on Windows EC2s
Now logging in to account B I can find the public document and choose to run it.
![](/image/doc.png)

I could also inspect the document, where the obfuscation in the content should alert me. There is a risk that a document you have previously inspected have had the version updated, or it could be that a vendor have shared a document with your account directly and you become a victim of supply chain attack through SSM. Luckily, the Antimalware Scan Interface (AMSI) for Windows blocks the first attempt and I can immediately see it in Systems Manager from the output it provides.

![](/image/ssmoutput.png)

Now there are [numerous ways](https://blog.f-secure.com/hunting-for-amsi-bypasses/) to bypass this and let's assume a mature attacker would have embedded this and move on to detection. For this demo I'll add a simple way to bypass this by just disabling Windows Defender on the line before. 

![](/image/ssm2.png)

It will then execute the stager successfully. 

![](/image/empire.png)

### Protecting against malicious documents
As a general best practice, you should avoid allowing the Systems Manager to execute commands as SYSTEM by [disabling its administrative permissions](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-ssm-user-permissions.html). You should also restrict which users can use the Run command and which assets they are allowed to use it for. For interactive sessions, you should [restrict which commands can be executed during sessions](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-restrict-command-access.html).

Whenever specifying your own document, make sure to include a hash and validate the file hash prior to execution. The hash can be stored as part of the [DocumentDescription](https://docs.aws.amazon.com/systems-manager/latest/APIReference/API_DocumentDescription.html)

Due to the inability to specify NotResource in a Service Control Policy, I have not yet found a way to whitelist a specific account, I am however able to blacklist accounts that publish SSM documents I want to avoid by denying the action. This approach probably doesn't scale but maybe someone know how I can turn this into a whitelist. 
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenySSMPublisher",
            "Effect": "Deny",
            "Action": [
                "ssm:SendCommand"
            ],
            "Resource": "arn:aws:ssm:us-east-1:454212039551:document/*"
        }
    ]
}
```
{: .language-json}

### Detecting execution of malicious documents
If you use AWS you hopefully already have a way of looking up which accounts you have in your Organization and that you trust, if you use that in combination with a SIEM you can alert whenever a document is executed from an account that your not in control of. CloudTrail will log the AccountId of the account that owns any document being run. 

![](/image/ssmdocct.png)

Any PowerShell documents executed through Systems Manager Run or Sessions will be logged by PowerShell,[Microsoft has written a great guide for PS logging](https://devblogs.microsoft.com/powershell/powershell-the-blue-team/). Common techniques for detecting exploitation through PowerShell will apply from here. 

You will have also have additional visibility through the following mechanisms:
- CloudTrail events name: SendCommand and StartSession
- CloudWatch logs for run command: https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-rc-setting-up-cwlogs.html
- SSM Agent Logs: https://docs.aws.amazon.com/systems-manager/latest/userguide/monitoring-ssm-agent.html
- Session Activity Logging: https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-logging.html

The combination of all these makes Session Manager and Run Command powerful tools that allows secure remote administration without having to use a bastion host.

Any other ideas for protecting or abusing SSM? Let me know. 