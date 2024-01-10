---
layout: post
title: 'Azure Update Manager pre and post events for turned off VMs'
date: 2024-01-10
logo: 'wrench'
description: This post explains how to implement pre and post events for VMs that are turned off for patching in Update Manager.
image: /_images/2024-01-10-update-manager-start-stop-vm-main.png
comments: true
---

## Introduction

[Azure Update Manager] is a quite new Azure service that went [Generally Available] 18th of September 2023. 

[Azure Update Manager]:https://learn.microsoft.com/en-us/azure/update-manager/
[Generally Available]:https://techcommunity.microsoft.com/t5/azure-governance-and-management/generally-available-azure-update-manager/ba-p/3928878

Azure Update Manager has been redesigned and doesn't depend on Azure Automation or Azure Monitor Logs, as required by the [Azure Automation Update Management] feature. 

[Azure Automation Update Management]:https://learn.microsoft.com/en-us/azure/automation/update-management/overview

Since the Azure Log Analytics agent, also known as the Microsoft Monitoring Agent (MMA) [will be retired] in August 2024, customers are recommended to move to [Azure Update Manager].

[will be retired]:https://learn.microsoft.com/en-us/azure/automation/update-management/overview

In [Azure Automation Update Management] there was a possibility to use [pre-scripts and post-scripts] to handle VMs that are turned off for cost saving purposes.

[pre-scripts and post-scripts]:https://learn.microsoft.com/en-us/azure/automation/update-management/pre-post-scripts

In the initial release of [Azure Automation Update Management] this possibility wasn't available. This recently got changed in December 2023, where [Pre and Post Events] got introduced for [Azure Update Manager] in preview.

[Pre and Post Events]:https://learn.microsoft.com/en-us/azure/update-manager/whats-new#pre-and-post-events-preview

In this post, I will explain how to implement pre and post events for VMs that are turned off for patching in [Azure Update Manager].

## Approach

The following approach is implemented to start/stop VMs in a Maintenance Configuration of Update Manager

![introduction](/_images/2024-01-10-update-manager-start-stop-vm-main.png)

1) The `Maintenance Configuration` about to be triggered *(T -20 minutes)* 

2) The `Maintenance Configuration` triggers an event to the Event Grid System Topic with the Event Type: `Microsoft.Maintenance.PreMaintenanceEvent`

3) The `Event Subscription` for `start vm` gets triggered, the `Event Subscription` is configured to pickup events with the Event Type: `Microsoft.Maintenance.PreMaintenanceEvent`, the event subscription is configured to trigger a webhook with the payload from the event.

4) The `Automation Webhook` for starting the start vm `runbook` gets triggered with the payload from the event.

5) The `Automation Runbook` for start vm gets triggered.

6) The `Automation Runbook` will get all the VMs and Arc Virtual Machines in scope and starts the deallocated or stopped ones. The runbook will tag the VMs that are started with the `UpdateManagerState tag` and `RunId value` for the maintenance configuration run.

7) The actual patching will start *(T -0 minutes)* 

8) After the configured time window for the patching, The `Maintenance Configuration` triggers an event to the Event Grid System Topic with the Event Type: `Microsoft.Maintenance.PostMaintenanceEvent` 

9) The `Event Subscription` for `stop vm` gets triggered, the `Event Subscription` is configured to pickup events with the Event Type: `Microsoft.Maintenance.PostMaintenanceEvent`, the event subscription is configured to trigger a webhook with the payload from the event.

10) The `Automation Webhook` for starting the stop vm `runbook` gets triggered with the payload from the event.

11) The `Automation Runbook` for stop vm gets triggered.

12) The `Automation Runbook` will get all the VMs and Arc Virtual Machines in scope and stops the started machines with the `UpdateManagerState tag` and `RunId value` for the maintenance configuration run. The tag will be removed after stopping the VMs and Arc Virtual Machines.

## Configuration

### User Assigned Managed Identity

[Create a User Assigned Managed Identity] `id-updatemanager` and assign the required roles. Store the `Client Id` for later

[Create a User Assigned Managed Identity]:https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp

![role assignments](/_images/2024-01-10-update-manager-start-stop-uami.png)

[Assign the following roles] to the `User Assigned Managed Identity` at `subscription` scope:

- **Reader** (To read out Azure Resource Graph for the Maintenance Runs) 
- **Virtual Machine Contributor** (To start/stop the VMs)
- **Tag Contributor** (To tag the VMs and identity when started)

[Assign the following roles]:https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal?tabs=delegate-condition

![role assignments](/_images/2024-01-10-update-manager-start-stop-subscription-role-assignments.png)


### Automation Account 

We will use a `Automation Runbook` to start all the VMs. The Runbook will get triggered from a `webhook` and will user the `User Assigned Managed Identity` to authenticate to Azure.

#### Automation Account Modules

The Automation Runbook requires the following `PowerShell Modules`:

- [ThreadJob] 
- [Az.ResourceGraph]

[ThreadJob]:https://learn.microsoft.com/en-us/powershell/module/threadjob/?view=powershell-7.4
[Az.ResourceGraph]:https://learn.microsoft.com/en-us/powershell/module/az.resourcegraph/?view=azps-11.1.0

You can import the modules using the [import module guide] 

[import module guide]:https://learn.microsoft.com/en-us/azure/automation/shared-resources/modules#import-az-modules

**[Note]** 
Make sure you select the Runtime version **7.2** for the Modules

#### Automation Account Variable

Create a `variable` with the name `var-clientid-id-updatemanager` that contains the `ClientId` from the `User Assigned Managed Identity` that you created earlier.

![automation variable](/_images/2024-01-10-update-manager-start-stop-automation-variable.png)

#### Automation Account Runbooks

There will be 2 runbooks deployed:

![preview feature](/_images/2024-01-10-update-manager-start-stop-automation-runbooks.png)

**[Note]** 
Make sure you select the Runtime version **7.2** for the PowerShell runbook

Here is the content of the scripts:

*UpdateManager-StartVMs*

``` powershell
param
(
    [Parameter(Mandatory=$false)]
    [object] $WebhookData
)

$miUpdateManager = Get-AutomationVariable -Name "var-clientid-id-updatemanager"

Connect-AzAccount -Identity -AccountId $miUpdateManager | Out-Null

$notificationPayload = $WebhookData.RequestBody | ConvertFrom-Json -depth 100
$maintenanceRunId = $notificationPayload[0].data.CorrelationId
$resourceSubscriptionIds = $notificationPayload[0].data.ResourceSubscriptionIds

if ($resourceSubscriptionIds.Count -eq 0) {
    Write-Output "Resource subscriptions are not present."
    break
}

Write-Output "Querying ARG to get machine details [MaintenanceRunId=$maintenanceRunId][ResourceSubscriptionIdsCount=$($resourceSubscriptionIds.Count)]"

$argQuery = @"
    maintenanceresources 
    | where type =~ 'microsoft.maintenance/applyupdates'
    | where properties.correlationId =~ '$($maintenanceRunId)'
    | where id has '/providers/microsoft.compute/virtualmachines/'
    | order by id asc
"@

Write-Output "Arg Query Used: $argQuery"

$allMachines = [System.Collections.ArrayList]@()
$skipToken = $null

do
{
    $res = Search-AzGraph -Query $argQuery -First 1000 -SkipToken $skipToken -Subscription $resourceSubscriptionIds
    $skipToken = $res.SkipToken
    $allMachines.AddRange($res.Data)
} while ($skipToken -ne $null -and $skipToken.Length -ne 0)

if ($allMachines.Count -eq 0) {
    Write-Output "No Machines were found."
    break
}

$jobIDs= New-Object System.Collections.Generic.List[System.Object]
$startableStates = "stopped" , "stopping", "deallocated", "deallocating"

$allMachines | ForEach-Object {
    $vmId = $_.properties.resourceId
    $split = $vmId -split "/";
    $subscriptionId = $split[2]; 
    $rg = $split[4];
    $name = $split[8];

    Write-Output ("Subscription Id: " + $subscriptionId)

    $mute = Set-AzContext -Subscription $subscriptionId
    $vm = Get-AzVM -ResourceGroupName $rg -Name $name -Status -DefaultProfile $mute

    $state = ($vm.Statuses[1].DisplayStatus -split " ")[1]
    if($state -in $startableStates) {
        Write-Output "Starting '$($name)' ..."

        $newJob = Start-ThreadJob -ScriptBlock { param($rg, $name, $sub) $context = Set-AzContext -Subscription $sub; Start-AzVM -ResourceGroupName $rg -Name $name -DefaultProfile $context} -ArgumentList $rg, $name, $subscriptionId
        $jobIDs.Add($newJob.Id)
        $tags = @{"UpdateManagerState"="$maintenanceRunId"}
        Update-AzTag -ResourceId $_.properties.resourceId -Tag $tags -operation Merge | Out-Null
    } else {
        Write-Output ($name + ": no action taken. State: " + $state) 
    }
}

$jobsList = $jobIDs.ToArray()
if ($jobsList)
{
    Write-Output "Waiting for machines to finish starting..."
    Wait-Job -Id $jobsList
}

foreach($id in $jobsList)
{
    $job = Get-Job -Id $id
    if ($job.Error)
    {
        Write-Output $job.Error
    }
}
```

*UpdateManager-StopVMs*

``` powershell
param
(
    [Parameter(Mandatory=$false)]
    [object] $WebhookData
)

$miUpdateManager = Get-AutomationVariable -Name "var-clientid-id-updatemanager"

Connect-AzAccount -Identity -AccountId $miUpdateManager | Out-Null

# Install the Resource Graph module from PowerShell Gallery
# Install-Module -Name Az.ResourceGraph

$notificationPayload = ConvertFrom-Json -InputObject $WebhookData.RequestBody
$maintenanceRunId = $notificationPayload[0].data.CorrelationId
$resourceSubscriptionIds = $notificationPayload[0].data.ResourceSubscriptionIds


if ($resourceSubscriptionIds.Count -eq 0) {
    Write-Output "Resource subscriptions are not present."
    break
}

Start-Sleep -Seconds 30
Write-Output "Querying ARG to get machine details [MaintenanceRunId=$maintenanceRunId][ResourceSubscriptionIdsCount=$($resourceSubscriptionIds.Count)]"

$argQuery = @"
    maintenanceresources 
    | where type =~ 'microsoft.maintenance/applyupdates'
    | where properties.correlationId =~ '$($maintenanceRunId)'
    | where id has '/providers/microsoft.compute/virtualmachines/'
    | order by id asc
"@

Write-Output "Arg Query Used: $argQuery"

$allMachines = [System.Collections.ArrayList]@()
$skipToken = $null

do
{
    $res = Search-AzGraph -Query $argQuery -First 1000 -SkipToken $skipToken -Subscription $resourceSubscriptionIds
    $skipToken = $res.SkipToken
    $allMachines.AddRange($res.Data)
} while ($skipToken -ne $null -and $skipToken.Length -ne 0)

if ($allMachines.Count -eq 0) {
    Write-Output "No Machines were found."
    break
}

$jobIDs= New-Object System.Collections.Generic.List[System.Object]
$stoppableStates = "starting", "running"

$allMachines | ForEach-Object {
    $vmId =  $_.properties.resourceId

    $split = $vmId -split "/";
    $subscriptionId = $split[2]; 
    $rg = $split[4];
    $name = $split[8];

    Write-Output ("Subscription Id: " + $subscriptionId)

    $mute = Set-AzContext -Subscription $subscriptionId
    $vm = Get-AzVM -ResourceGroupName $rg -Name $name -Status -DefaultProfile $mute

    $state = ($vm.Statuses[1].DisplayStatus -split " ")[1]
    if($state -in $stoppableStates) {
        Write-Output "Stopping '$($name)' ..."

        $newJob = Start-ThreadJob -ScriptBlock { param($resource, $vmname, $sub) $context = Set-AzContext -Subscription $sub; Stop-AzVM -ResourceGroupName $resource -Name $vmname -Force -DefaultProfile $context} -ArgumentList $rg, $name, $subscriptionId
        $jobIDs.Add($newJob.Id)
        $tags = @{"UpdateManagerState"="$maintenanceRunId"}
        Update-AzTag -ResourceId $_.properties.resourceId -Tag $tags -operation Merge | Out-Null
    } else {
        Write-Output ($name + ": no action taken. State: " + $state) 
    }
}

$jobsList = $jobIDs.ToArray()
if ($jobsList)
{
    Write-Output "Waiting for machines to finish stop operation..."
    Wait-Job -Id $jobsList
}

foreach($id in $jobsList)
{
    $job = Get-Job -Id $id
    if ($job.Error)
    {
        Write-Output $job.Error
    }
}
```

#### Automation Account Webhooks

[Create a webhook] for both runbooks.

[Create a webhook]:https://learn.microsoft.com/en-us/azure/automation/automation-webhooks?tabs=portal#create-a-webhook

Keep the URLs from the webhooks for later (it will be the only time to display)

Result:

*start vm* webhook

![webhook start vm](/_images/2024-01-10-update-manager-start-stop-automation-wh-start-vm.png)

*stop vm* webhook

![webhook stop vm](/_images/2024-01-10-update-manager-start-stop-automation-wh-stop-vm.png)

### Register your subscription for public preview

Follow this guide: [Register your subscription for public preview]

[Register your subscription for public preview]:https://learn.microsoft.com/en-us/azure/update-manager/manage-pre-post-events#register-your-subscription-for-public-preview

This will enable the preview feature `Pre and Post Events` on your subscription.

![preview feature](/_images/2024-01-10-update-manager-start-stop-vm-preview-feature.png)

### Maintenance Configuration
Setup a [Maintenance Configuration for Update Manager] for In guest patching.

[Maintenance Configuration for Update Manager]:https://learn.microsoft.com/en-us/azure/update-manager/scheduled-patching?tabs=schedule-updates-single-machine%2Cschedule-updates-scale-overview

### Event Grid Topic

Per schedule an `Event Grid topic` must be setup.
The `Event Grid Topic` can be shared for multiple subscriptions. 

#### Event Grid Subscriptions

Here's how to create an Event Grid subscription on the Maintenance Configuration Event with a filter on `Microsoft.Maintenance.PreMaintenanceEvent` to start the VM.

On the selected Maintenance configuration page, under Events, select `Web Hook`.  

![event main](/_images/2024-01-10-update-manager-start-stop-vm-event-create.png)

In the `Create Event Subscription` wizard, fill in the following:

- EVENT SUBSCRIPTION DETAILS
  - name: *[name of the Event Subscription]* (Example: *start-vm*) 
- TOPIC DETAILS
  - System Topic Name: *[the name of the System Topic]* (Example: *egst-start-stop-demo*)
- EVENT TYPES
  - Filter to Event Types: *Pre Maintenance Event* (This filters out the Pre Maintenance Event)
- ENDPOINT DETAILS
  - Endpoint for the webhook: *[your endpoint]* (This is the URLs from the webhook you created earlier)
  
![event start](/_images/2024-01-10-update-manager-start-stop-vm-event-create-start-vm.png)

Repeat the `Create Event Subscription` wizard for the *stop-vm* `Event Subscription` and select the *Post Maintenance Event* event type.

## Result

As result of the configuration, the Maintenance Configuration will: 

- Start the Virtual Machines that are turned off using an Automation Runbook

![start vm job](/_images/2024-01-10-update-manager-start-stop-automation-job-start-vm.png)

- Stop the Virtual Machines that are turned off using an Automation Runbook

![stop vm job](/_images/2024-01-10-update-manager-start-stop-automation-job-stop-vm.png)

