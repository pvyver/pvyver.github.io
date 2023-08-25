---
layout: post
title: 'Deployment stacks'
date:   2023-08-25 
logo: 'fa fa-code'
comments: true
---

<img src="/_images/2023-08-25-deployment-stacks_deployment-stack-icon.png">

## Table of contents

## Introduction

### Deployment states

There are multiple `Infrastructure as Code` languages (Terraform, Pulumi, Bicep, ARM).

Depending on the language used, there is a specific way of holding the state of resources from a deployment:
- Terrraform has its state file that holds the configuration
- Pulumi has its state backend 
- Bicep & ARM have Azure as the real state of the resource

The state has always been a weakness of Bicep and ARM. The lifecycle of the resources where dificult to track, you could implement workarounds with [what-if operations] always an additional step to take before a deployment.

[what-if operations]:https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-what-if?tabs=azure-powershell%2CCLI

### Deployment Stacks

Microsoft recently anounced a **public preview** of [ARM Deployment Stacks] to tackle this. To me this was the missing element in order to use Bicep or ARM templates.

[ARM Deployment Stacks]:https://techcommunity.microsoft.com/t5/azure-governance-and-management/arm-deployment-stacks-now-public-preview/ba-p/3871180

Bicep or ARM template files can be submitted to a Deployment Stack, the Deployment Stacks basically define the resources that are managed by the stack.

#### Creating / Updating Deployment Stacks

Deployment Stacks can be created at the following scopes:
- Resource Group
- Subscription
- Management Group

Creating/Update Deployment Stacks PowerShell & CLI commands:
- **Resource Group**
  - PowerShell: `
    - [New-AzResourceGroupDeploymentStack] (new) 
    - [Set-AzResourceGroupDeploymentStack] (update)
  - CLI: [az stack group create] (new or update)
- **Subscription** 
  - PowerShell: 
    - [New-AzSubscriptionDeploymentStack] (new) 
    - [Set-AzSubscriptionDeploymentStack] (update)
  - CLI: [az stack group create] (new or update)
- **Management Group**
  - PowerShell: 
    - [New-AzResourceGroupDeploymentStack] (new) 
    - [Set-AzManagementGroupDeploymentStack] (update)
  - CLI: [az stack group create] (new or update)

[New-AzResourceGroupDeploymentStack]:https://learn.microsoft.com/en-us/powershell/module/az.resources/new-azresourcegroupdeploymentstack?view=azps-10.1.0

[Set-AzResourceGroupDeploymentStack]:https://learn.microsoft.com/en-us/powershell/module/az.resources/set-azresourcegroupdeploymentstack?view=azps-10.2.0

[New-AzSubscriptionDeploymentStack]:https://learn.microsoft.com/en-us/powershell/module/az.resources/new-azsubscriptiondeploymentstack?view=azps-10.2.0

[Set-AzSubscriptionDeploymentStack]:https://learn.microsoft.com/en-us/powershell/module/az.resources/set-azsubscriptiondeploymentstack?view=azps-10.2.0

[New-AzResourceGroupDeploymentStack]:https://learn.microsoft.com/en-us/powershell/module/az.resources/new-azresourcegroupdeploymentstack?view=azps-10.2.0

[Set-AzManagementGroupDeploymentStack]:https://learn.microsoft.com/en-us/powershell/module/az.resources/set-azmanagementgroupdeploymentstack?view=azps-10.2.0

[az stack group create]:https://learn.microsoft.com/en-us/cli/azure/stack/group?view=azure-cli-latest#az-stack-group-create


> **Note:** You can **Block Unwanted Changes** by setting a `DenySettingsMode` on the Deployment Stack (*None, DenyDelete, DenyWriteAndDelete*)

#### Deleting Deployment Stacks

Delete Deployment Stacks PowerShell & CLI commands:

- **Resource Group**
  - PowerShell: [Remove-AzResourceGroupDeploymentStack]
  - CLI: [az stack group delete]
- **Subscription** 
-   - PowerShell: [Remove-AzSubscriptionDeploymentStack]
    - CLI: [az stack sub delete]
- **Management Group**
-   - PowerShell: [Remove-AzManagementGroupDeploymentStack]
    - CLI: [az stack mg delete]

[Remove-AzResourceGroupDeploymentStack]:https://learn.microsoft.com/en-us/powershell/module/az.resources/remove-azresourcegroupdeploymentstack?view=azps-10.2.0
[Remove-AzSubscriptionDeploymentStack]:https://learn.microsoft.com/en-us/powershell/module/az.resources/remove-azsubscriptiondeploymentstack?view=azps-10.2.0
[Remove-AzManagementGroupDeploymentStack]:https://learn.microsoft.com/en-us/powershell/module/az.resources/remove-azresourcegroupdeploymentstack?view=azps-10.2.0

[az stack group delete]:https://learn.microsoft.com/en-us/cli/azure/stack/group?view=azure-cli-latest#az-stack-group-delete
[az stack sub delete]:https://learn.microsoft.com/en-us/cli/azure/stack/sub?view=azure-cli-latest#az-stack-sub-delete
[az stack mg delete]:https://learn.microsoft.com/en-us/cli/azure/stack/mg?view=azure-cli-latest#az-stack-mg-delete

> **Note:** A **Cleanup** can be done after removal of a Deployment Stack (Resources or/and Resource Groups), this is optional flag by default Resources or/and Resource Groups are detached from a Deployment Stack. Possible `flags` for Deploymnt Stack deletation:
(*DeleteAll, DeleteResourceGroups ,DeleteResources*). 

## Deployment Stack - Subscription Example

In this example, the template will deploy:
- a resource group
- dynamic windows function app (with all contained resources)

The Deployment Stack will be scoped at subscription level.

### Bicep Template & Parameter files

`bicepdemo1.bicep`

``` bicep 
targetScope = 'subscription'

// **Parameters**
// Generic Parameters - Used in multiple modules 
param parLocation string
param parLocationAbbrevation string
param parEnvironment string
param parPurpose string

resource resResourceGroup 'Microsoft.Resources/resourceGroups@2022-09-01' = {
  name: 'rg-${parLocationAbbrevation}-${parEnvironment}-${parPurpose}'
  location: parLocation
}

// module dynamic windows function app
module modFunctionAppWindowsConsumption '../../modules/functionAppWindowsConsumption/functionAppWindowsConsumption.bicep' = {
  name: 'deploy-fun-${parLocationAbbrevation}-${parEnvironment}-${parPurpose}'
  scope: resourceGroup(resResourceGroup.name)
  params: {
    functionAppName: 'fun-${parLocationAbbrevation}-${parEnvironment}-${parPurpose}'
    functionWorkerRuntime: 'dotnet'
    location: parLocation
    storageAccountName: 'st${parLocationAbbrevation}${parEnvironment}${parPurpose}'
    storageAccountType: 'Standard_LRS'
    applicationInsightsName: 'appi-${parLocationAbbrevation}-${parEnvironment}-${parPurpose}'
    hostingPlanName: 'asp-${parLocationAbbrevation}-${parEnvironment}-${parPurpose}'
  }
}

```

`tst-bicepdemo1.bicepparam`
``` cs
using '../bicepdemo1.bicep'

param parLocation            = 'westeurope'
param parLocationAbbrevation = 'weu'
param parEnvironment         = 'tst'
param parPurpose             = 'bicepdemo1'
```
### Deploy a Deployment Stack

`New-Deploymentstack.ps1`

``` cs
$inputObject = @{
    Name                        = "stack-weu-tst-bicepdemo1"
    TemplateFile                = "infra-as-code/bicep/orchestration/bicepdemo1/bicepdemo1.bicep"
    TemplateParameterFile       = "infra-as-code/bicep/orchestration/bicepdemo1/parameters/tst-bicepdemo1.bicepparam"
    DenySettingsMode            = "DenyWriteAndDelete"
    Location                    = "westeurope"
}

New-AzSubscriptionDeploymentStack  @inputObject -Force
```

#### Deployment Result

Ar subscription level, the Deployment Stack will become visible:
<img src="/_images/2023-08-25-deployment-stacks_deployment-stack-subscription.png">

In the deployed Deployment Stack, the details will be displayed:
<img src="/_images/2023-08-25-deployment-stacks_deployment-stack-subscription-overview.png">

> **Note:** The `Deny status` for all Resource Groups and Resources of the Deployment Stack are set to `denyWriteAndDelete`, the Deployment Stack will block all write and delete actions on the Resource Group or Resources controlled by the Deployment Stack.

Deny assignments:
<img src="/_images/2023-08-25-deployment-stacks_deployment-stack-subscription-denyassignments.png">

In an attemnt of deleting a storage account controlled by the Deployment Stack, will result as a `deny` because of the `deny assignment` created by Deployment Stack:
<img src="/_images/2023-08-25-deployment-stacks_deployment-stack-subscription-denyassignments-deletesta.png">

### Update a Deployment Stack

To update a Deployment Stack (update controlled Resource Group, Resources or update DenySettingsMode), you can use the following script.

> **Note:** In this example the `DenySettingMode` is set to `None`

`Set-Deploymentstack.ps1`

``` cs
$inputObject = @{
    Name                        = "stack-weu-tst-bicepdemo1"
    TemplateFile                = "infra-as-code/bicep/orchestration/bicepdemo1/bicepdemo1.bicep"
    TemplateParameterFile       = "infra-as-code/bicep/orchestration/bicepdemo1/parameters/tst-bicepdemo1.bicepparam"
    DenySettingsMode            = "None"
    Location                    = "westeurope"
}

Set-AzSubscriptionDeploymentStack  @inputObject -Force 
```

#### Update Result

The `Deny status` for the resources were updated:

<img src="/_images\2023-08-25-deployment-stacks_deployment-stack-subscription-overview-update-denystatus.png">

Also visible in the `Deny assignments` blade:

<img src="/_images/2023-08-25-deployment-stacks_deployment-stack-subscription-update-denyassignments.png">

### Delete a Deployment Stack

To delete a Deployment Stack **and delete the resource groups and resources** you can use the following script.

`Remove-Deploymentstack.ps1`

``` cs
Remove-AzSubscriptionDeploymentStack `
    -name "stack-weu-tst-bicepdemo1" `
    -DeleteAll `
    -Force
```

> **Note:** The removal of Deployment Stacks support the following options:
>
> Detach: Detach resources and resource groups (Default)
> 
> DeleteAll: Delete both the resources and the resource groups
> 
> DeleteResourceGroups: Delete the resource groups only
> 
> DeleteResources: Delete resources only
