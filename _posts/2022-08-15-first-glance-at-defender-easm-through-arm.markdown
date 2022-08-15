---
layout: post
title:  "First glance at Defender EASM through ARM and Azure Bicep"
date:   2022-08-15
---
I haven’t been working with Microsoft Azure for 5 years but tried keeping up to date, which I am grateful for now that I am embarking on a new journey which will increase my usage of Azure. More on the journey later..

To enhance my learning through writing about it, I decided to create a summary of what I discovered while trying to deploy External Attack Surface Management (EASM), which is a *not yet supported* service through Azure Resource Manager (ARM). 

My first discovery while starting to use Azure after 5 years is that Bicep is now the way to go for efficiently writing Infrastructure as Code (IAC), and I had the perfect case to learn exactly how extensible it can be. 

I made sure to update my CLI and Bicep installation, where I learned that it does not have linter support for EASM, which is fair. Nor does it have CLI support yet from what I could find in the reference.

# Discovering resource provider and deploying

I went on and deployed EASM through the GUI which allowed me to extract the resource provider by running *az resource list -g rg-name*

![https://user-images.githubusercontent.com/26272119/184663375-f0e2fb13-d94c-4896-8af7-58800674c893.png](https://user-images.githubusercontent.com/26272119/184663375-f0e2fb13-d94c-4896-8af7-58800674c893.png)

```nasm
Microsoft.Easm/workspaces@2022-04-01-preview'
```

With the resource name, I knew I could easily make an ARM template, and knowing that Bicep is just an abstraction I can also create a template and save time on authoring the template.

```ruby
param name string = 'easm-work'
param region string = 'eastus'
resource Easm 'Microsoft.Easm/workspaces@2022-04-01-preview' = {
name: name
location: region
properties: {}
}
```

While creating the Bicep template I am warned that the Resource type does not have types available, so what’s great about Bicep is that it can easily warn me through connecting to an API, nice!

![https://user-images.githubusercontent.com/26272119/184665983-ac7d9250-a9af-440c-970f-4f9d698f94e9.png](https://user-images.githubusercontent.com/26272119/184665983-ac7d9250-a9af-440c-970f-4f9d698f94e9.png)

When something on my laptop makes an external connection, I become curious. A quick glance showed DNS requests to [onedscolprdwus08.westus.cloudapp.azure.com](http://onedscolprdwus08.westus.cloudapp.azure.com/) from my terminal, I didn’t dig further for now. 

After a running the *az bicep build* command I get my ARM template:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "metadata": {
    "_generator": {
      "name": "bicep",
      "version": "0.9.1.41621",
      "templateHash": "10012041125269374402"
    }
  },
  "parameters": {
    "name": {
      "type": "string",
      "defaultValue": "twowork"
    },
    "region": {
      "type": "string",
      "defaultValue": "eastus"
    }
  },
  "resources": [
    {
      "type": "Microsoft.Easm/workspaces",
      "apiVersion": "2022-04-01-preview",
      "name": "[parameters('name')]",
      "location": "[parameters('region')]",
      "properties": {}
    }
  ]
}
```

And now that I have an ARM template I can simply deploy it using the CLI:

```bash
az deployment group create --resource-group "easm-rg" --template-file ./easm.json
```

**Resource successfully created!** 

For now, it doesn’t seem like I can manage any of the settings within my EASM deployment through code, I’m crossing my fingers that works by the time the Resource Provider is officially supported. In the meantime, [Marius Sandbu has created a great guide for getting started with EASM](https://msandbu.org/getting-started-with-microsoft-defender-easm-external-attack-surface-management/) that I highly recommend reading.

# What I learned

I did this mostly as a quick way to learn more about Azure and ARM while also staying true to my principle of always defining things ‘as Code’. I figured the learning experience could be useful for others and decided to provide a summary of the process and my learnings.

- Azure Bicep performs a lookup to validate the schema prior to generating a template.
- ARM Templates can easily be extracted for new resources or discovered and supports deployment prior to it being documented
- Just like with Amazon Web Services, new services aren’t fully supported upon release.
