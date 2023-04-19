---
layout: post
title: Turning Azure virtual machines on and off from a GitHub action
date: 2023-04-19T00:00:00.0000000+02:00
categories: []
tags:
- Azure
- GitHub Actions
- Virtual Machines
- YAML
- DevOps
featuredImageUrl: https://LocalJoost.github.io/assets/2023-04-19-Turning-Azure-virtual-machines-on-and-off-from-a-GitHub-action/topright.png
comment_issue_id: 443
---
For building Mixed Reality applications I use an Azure virtual machine, because reasons. The great thing about virtual machines is they can be deployed pretty easily without having to order real hardware - the *problem* with virtual machines is they can rack up quite an Azure bill when left unattended, and although my friends at Microsoft won't mind, my credit card certainly will. 

Recentely I attended a session by [Barbara Forbes](https://twitter.com/ba4bes), on [FurtureTech 2023](https://futuretech.nl/), where she showed you could use a credential that allowed a GitHub action to login into an Azure tenant. It was actually a very tiny part of the talk, but it stuck in my mind: since I already knew you could startup and shutdown a virtual machine using the Azure Cloud Shell, I wondered: would it be possible to do that from a GitHub action as well? And yes indeed, you can.

## Getting the login credential

When you select your virtual machine in the Azure portal, you will see a whopping long URL in your browser's address bar. Something like this:

portal.azure.com/#@somenamehere.onmicrosoft.com/resource/subscriptions/<br>
subscriptionid/resourceGroups/VMresourcegroup/providers/Microsoft.Compute/<br>
virtualMachines/VMname/overview

You will need this part right away:

**subscriptionid/resourceGroups/VMresourcegroup**

and the **VMName** later

Open a cloud shell with this thingy top right in the browser:
![](/assets/2023-04-19-Turning-Azure-virtual-machines-on-and-off-from-a-GitHub-action/topright.png)

And enter the following command:

```powershell
az ad sp create-for-rbac --name "somenamehere" --role contributor \
    --scopes /subscriptions/subscriptionid/resourceGroups/VMresourcegroup \
    --sdk-auth
```

This will respond with something very much like this:
```javascript
{
  "clientId": "{someguid}",
  "clientSecret": "{someguid}",
  "subscriptionId": "{someguid}",
  "tenantId": "{someguid}",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```
This entire things needs to be stored in a GitHub Actions *secret*. I called it AZURE_CREDENTIALS

![](/assets/2023-04-19-Turning-Azure-virtual-machines-on-and-off-from-a-GitHub-action/secret.png)

## Stopping the virtual machine manually and on a schedule

This requires this simple bit of YAML. The first step logs in, the second step stops the virtual machine.

```yaml
name: stopvm

on:
  workflow_dispatch:
  schedule:
    - cron: '15 15 * * 1-5'
    - cron: '19 15 * * 1-5'

jobs:
  deploy:
    runs-on: ubuntu-latest

    name: shutdown
    steps: 

      - name: Azure Login
        uses: Azure/login@v1
        with:
          creds: ${% raw %}{{ secrets.AZURE_VM_CREDENTIALS }}{% endraw %}  

      - name: Azure CLI script
        uses: azure/CLI@v1
        with:
          azcliversion: 2.30.0
          inlineScript: |
            az vm stop --resource-group VMresourcegroup --name VMname
            az vm deallocate --resource-group VM resource group --name VMname
```

All times are in UTC, so the build server is stopped at 3:15pm and at 7:15pm another attempt is done. In CEST we are now in DST so that's 5:15pm and 9:15pm. You can also stop it manually.

Note: the virtual machine is not only stopped, but also *deallocated*. Otherwise, it will still be billed.

## Starting the virtual machine on demand

Now this is the really funky thing. Initially, I just let the machine start at 8am. But sometimes days go by without a branch being merged to develop. So why not have the virtual machine start *on demand*? Turned out to be pretty easy. On top of my build file I added this extra job:

```yaml
name: mybuild

on:
  push:
      branches:
        - develop

jobs:
  # First, start the vm
  startvm:
    runs-on: ubuntu-latest

    name: startvm
    steps: 

      - name: Azure Login
        uses: Azure/login@v1
        with:
          creds: ${% raw %}{{ secrets.AZURE_VM_CREDENTIALS }}{% endraw %}

      - name: Azure CLI script
        uses: azure/CLI@v1
        with:
          azcliversion: 2.30.0
          inlineScript: |
            az vm start --resource-group VMresourcegroup --name VMname

  # Then, build the project

  build-project:
    runs-on: somerunner
```

And now the build server spins up whenever the first merge of the day is coming in. Technically, I could have it shutdown again afterwards and *really* pinch down on virtual machine costs, but since merges tend to come in flocks, that's not very practical so I decided against it. But it would be prefectly possible. But I did add two stop moments, in case a merge happens after 5:15. 

Since you can copy and paste all the code off this blog post, I will dispense with an actual demo project this time.