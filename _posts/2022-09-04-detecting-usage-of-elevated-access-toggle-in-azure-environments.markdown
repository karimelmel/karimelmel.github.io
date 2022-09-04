---
layout: post
title:  "Detecting usage of Elevated Access Toggle in Azure environments"
date:   2022-09-04
---
As I’ve started looking into Microsoft Azure, one of the risks I’ve identified is that a [Global Administrator can elevate access to manage all Azure subscriptions and management groups](https://docs.microsoft.com/en-us/azure/role-based-access-control/elevate-access-global-admin), this technique is also covered in the [Azure Threat Research Matrix as AZT402.](https://microsoft.github.io/Azure-Threat-Research-Matrix/PrivilegeEscalation/AZT402/AZT402/) My first instinct would be to eliminate this risk completely, which unfortunately is not possible. Global Admins should be rare and well protected, but is still a risk of interest.

Given the risk cannot be completely mitigated, my next instinct is to look for a detective approach. My assumption that the Management Group logs would be part of the Azure Activity logs by default was quickly proven wrong. 

I came across two blog posts from @samilamppu that covers two ways of extracting the logs to detect the specific scenario:

- [https://samilamppu.com/2022/04/08/detect-elevate-access-activity-in-azure-with-microsoft-sentinel-native-capabilities/](https://samilamppu.com/2022/04/08/detect-elevate-access-activity-in-azure-with-microsoft-sentinel-native-capabilities/)
- [https://samilamppu.com/2021/02/09/monitor-elevate-access-activity-in-azure-with-azure-sentinel/](https://samilamppu.com/2021/02/09/monitor-elevate-access-activity-in-azure-with-azure-sentinel/)

The problem here is that the data relies on another source, Microsoft Defender for Cloud Apps which all Azure customers may not have due to license constraints.

I did some digging into the API and came across the [API for Management Group Diagnostic Settings](https://docs.microsoft.com/en-us/rest/api/monitor/management-group-diagnostic-settings).  Great!

To enable it on the Tenant Root Level (top-level management group), all I would have to do is run the following one-liner followed by payload:

```bash
az rest --method put --uri "https://management.azure.com/providers/microsoft.management/managementGroups/{TENANTID OR MANAGEMENT GROUP ID}/providers/microsoft.insights/diagnosticSettings/{Name}?api-version=2020-01-01-preview" --body "@body.json"
```

```bash
{
    "properties": {
        "workspaceId": "/subscriptions/{SubcriptionID}/resourceGroups/{ResourceGroupName}/providers/Microsoft.OperationalInsights/workspaces/{WorkspaceName}",
        "logs": [
        {
            "category": "Administrative",
            "enabled": true
        },
        {
            "category": "Policy",
            "enabled": true
        }
        ]
  }
}
```

Now you can navigate to the Workspace and start seeing events with the Hierarchy of your Tenant ID with a 10 minute delay. To filter out values from a top-level Management Group you can use the query below:

```bash
AzureActivity
| where Hierarchy == "<TENANT ID"
```

The specific behaviour you are looking to detect is whenever a Global Admin enables the setting below:

![image](https://user-images.githubusercontent.com/26272119/188316383-06db1600-f173-49c3-9293-68d12183da64.png)

To filter out that behaviour, run the query below in Log Analytics. This can also be the basis for your detection:

```bash
AzureActivity
| where parse_json(tostring(parse_json(Authorization).evidence)).role == "User Access Administrator"
| where OperationNameValue == "MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE"
```

And the data returned contains the specific event:
![image](https://user-images.githubusercontent.com/26272119/188316398-40bb0e54-c90b-41d2-84ae-387006a3c86c.png)

This is in my opinion the simplest way of enabling logging on the Management Group level without depending on additional automation. Keep in mind that a Global Admin can potentially take control of Azure resources through other tactics and this only covers this particular technique.