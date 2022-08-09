# Security considerations for hosting domain controllers in cloud

While Domain Controllers might not be something that new organizations, those born in the cloud would rely on today, it seems common for most existing organizations that seek to migrate.

Through my years of working, I’ve always had respect for Domain Controllers and the value it possess both from an availability perspective but more importantly when it comes to integrity and confidentiality. For the purpose of this post I will seek to cover Amazon Web Services and Microsoft Azure primarily, the same principles applies to Google Cloud Platform.

## The common landing zone

Below is an example of an AWS Landing Zone the illustration is the standard Landing Zone design and where there normally is nothing wrong in the design, however,  I have introduced a problem I tend to come across. **The Domain Controller is hosted in the “Shared Services Account”.** 

![https://user-images.githubusercontent.com/26272119/183304543-10ae5a89-0dab-4d00-9d18-ddf6ad5d3380.jpg](https://user-images.githubusercontent.com/26272119/183304543-10ae5a89-0dab-4d00-9d18-ddf6ad5d3380.jpg)

## Understanding the problem

If you don’t know why you should protect your Active Directory domain and/or Domain Controllers, I suggest you start [reading some of the content written by Sean Metcalf over the years](https://adsecurity.org/?p=1684), for this post I will assume you know enough about Active Directory to understand why it should be protected at a high cost.

While cloud provides a solid set of tools for efficiently managing infrastructure, a few things often seem forgotten. The most important being: **#1 A privileged user in the cloud can execute code on any EC2 or Virtual Machine through the ‘cloud provider API’ by default.**

In AWS and Azure, there are numerous ways to do this, such as:

- Modifying the integrity of files using the EBS API (AWS Only)
- Modifying the userdata and rebooting the instance
- Using AWS Systems Manager or Run Command

As a result of this, a full compromise of the environment is then a lot more likely if Active Directory is used for identities or computer objects, as a privileged user in the Shared Services account can likely elevate privileges to access other resources.

**#2 Data can be read from volumes directly or through snapshots**

- Creating a snapshot to read data from it directly through the EBS Direct Access API (AWS)
- Creating a snapshot and sharing with another account or reading directly within the account
- Accessing backups through Azure Backup or AWS Backup (hopefully, these are protected with strict controls)

There are even automated tools for this, below is an illustration on how the tool [‘CloudCopy’](https://github.com/Static-Flow/CloudCopy) can perform such an attack automated, with the permission to read and share a snapshot. This tool also have support for Azure. 

![https://user-images.githubusercontent.com/26272119/183359992-cfd4401a-5c00-46c9-95f3-225ec840c49f.png](https://user-images.githubusercontent.com/26272119/183359992-cfd4401a-5c00-46c9-95f3-225ec840c49f.png)

## Defending Domain Controllers in the cloud

To defend against a general attack on systems, the principle of least privilege should apply. For sensitive workloads that can be misused to attack the entire environment, other considerations should apply, a more ideal architecture could look something like the below in AWS, where you  should consider to have a separate Organization or use policies to restrict Organization-wide roles and have the workloads completely separate and only accessible from a Secure Access Workstation. If you chose to use Federation towards the ‘Secure Organization’, it should have a separate IdP that is inaccessible to the production environment. 

![https://user-images.githubusercontent.com/26272119/183308238-a6fbb36a-923c-4cce-bb62-26a9eb3a84f4.jpg](https://user-images.githubusercontent.com/26272119/183308238-a6fbb36a-923c-4cce-bb62-26a9eb3a84f4.jpg)

This design would adhere with [Microsoft’s “Enhanced Security Administrative Environment"](https://docs.microsoft.com/en-us/security/compass/esae-retirement) and could easily be translated to Azure with a separate Azure Active Directory tenant, Management Group and Subscription.

### Protecting Code Execution

For protecting code execution, the architecture above doesn’t do it by itself, you should also consider what Agents with code execution capabilities are installed by default on a virtual machine in public cloud, both Azure and AWS adds agents that are pre-built into the operating system allowing code execution, [there is a public list keeping track of these.](https://github.com/wiz-sec/cloud-middleware-dataset) 

### Protecting Snapshots

Technically, you could also protect towards Snapshots accessing the data [by leveraging host-level encryption](https://aws.amazon.com/blogs/aws/amazon-ec2-now-supports-nitrotpm-and-uefi-secure-boot/) as the API would be unable to decrypt the data. Disk-level encryption is not sufficient to prevent this or to protect against backup solutions that has guest-OS level access.

Another protection mechanism that could further complicate the extraction method for snapshots, would be by limiting the access to Snapshots through a [Private Endpoint](https://docs.microsoft.com/en-us/azure/virtual-machines/disks-enable-private-links-for-import-export-portal) in Azure, for AWS you can also create VPC Endpoint to restrict EBS Direct APIs, but in both cases you could use Azure Policy or a Service Control Policy to remove the ability to take a snapshot.

Note that although Azure Disk Encryption leverages Guest OS encryption, if you choose to enable it during VM provisioning it is done through code execution on the virtual machine through the Azure API and the agent is a pre-requisite. An alternative would be to manually enable this.

## Summary

I don’t think there is a definite answer to how Domain Controllers should be configured or segmented while running on virtual machines in cloud, but I do want to list what I believe is some fundamental best practices if you want to reduce the risk of a full-blown compromise or if you are following Microsoft’s guidance for ESAE aka Red Forest:

- Carefully segment Domain Controllers into a separate Account and/or Organization in AWS or Enterprise / Management Group and Subscriptions in Microsoft Azure.
- Consider a separate IdP for accessing cloud or restrict who can modify access to federation similar to domain admins.
- Avoid having Guest OS agents on domain controllers, so that Run command and similar services cannot be used for code execution.
- Use VPC endpoints/Private Endpoints for cloud services the server must interact with.
- Carefully threat model the implication of each service you allow a domain controller to access.
- Prohibit sharing of snapshots or carefully protect them within the same boundary as Domain Controllers. Only allow Snapshots to be accessed through a Private Endpoint in Azure and restrict any sharing in AWS.
- Implement a Secure Access Workstation to access highly sensitive environments