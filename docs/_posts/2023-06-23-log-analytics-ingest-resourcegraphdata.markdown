---
layout: post
title: 'Ingest Azure resource data into a Log Analytics Workspace Custom Table'
date:   2023-06-23 
logo: 'calculator'
comments: true
---

## Introduction

Creating `Log Analytics workspace Log queries` sometimes requires data from resources that is not available in any `Log Analytics workspace` table. For example: tags, resource properties, ...

As workaround, I developed a `Logic App` that queries `Azure Resource Graph` and injects the Logs in a `Log Analytics workspace` using the `Azure Monitor` [Data Collector API]:

[Data Collector API]:https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-ingestion-api-overview

![Flow](/_images/2023-06-23-log-analytics-ingest-resourcegraphdata-resources2log.png)

### Flow
1. Query the `Azure Resource Graph` for all `Azure Resources` (interval 15 minutes)
2. Post the retreived `json` data from all `Azure Resources` to the `Data Collection Endpoint`
3. Use the `Data Collection Rule` to transform the data and stream to a `Log Analytics workspace  Custom Table`

## The Setup

### Log Analytics workspace Resources_CL Table

<img src="/_images/2023-06-23-log-analytics-ingest-resourcegraphdata-lawtable.png" width="50">

In order to ingest data we have to setup a `Log Analytics workspace` custom table with a predefined schema. The fields more or less match the fields from the `Azure Resource Graph Resources`, but some of them are not allowed to be used in a custom `Log Analytics workspace Table`

Here is the schema I used to create the table:

`table.json`
``` json
{
    "properties": {
        "schema": {
            "name": "Resources_CL",
            "columns": [
                {
                    "name": "TimeGenerated",
                    "type": "datetime",
                    "description": "The time at which the data was generated"
                },
                {
                    "name": "extendedLocation",
                    "type": "dynamic"
                },
                {
                    "name": "identity",
                    "type": "dynamic"
                },
                {
                    "name": "location",
                    "type": "dynamic"
                },
                {
                    "name": "managedBy",
                    "type": "dynamic"
                },
                {
                    "name": "name",
                    "type": "string"
                },
                {
                    "name": "plan",
                    "type": "dynamic"
                },
                {
                    "name": "properties",
                    "type": "dynamic"
                },
                {
                    "name": "resourceGroup",
                    "type": "dynamic"
                },
                {
                    "name": "resourceId",
                    "type": "string"
                },
                {
                    "name": "resourceKind",
                    "type": "dynamic"
                },
                {
                    "name": "resourceTenantId",
                    "type": "string"
                },
                {
                    "name": "resourceType",
                    "type": "string"
                },
                {
                    "name": "sku",
                    "type": "dynamic"
                },
                {
                    "name": "subscriptionId",
                    "type": "string"
                },
                {
                    "name": "tags",
                    "type": "dynamic"
                },
                {
                    "name": "zones",
                    "type": "dynamic"
                }
            ]
        }
    }
}
```

There are [a few ways to create a Log Analytics Custom Table] using the Portal, API, CLI or PowerShell.

[a few ways to create a Log Analytics Custom Table]:https://learn.microsoft.com/en-us/azure/azure-monitor/logs/create-custom-table?tabs=azure-powershell-1%2Cazure-portal-2%2Cazure-portal-3

For this example I used the `PowerShell` way with the `Invoke-RestMethod`

``` powershell
# variables
$resourceGroup = "<your resource group>"
$logAnalyticsworkspaceId = "<your Log Analytics workspace Resource Id>"
$tableName = "Resources_CL"

function get-azCachedAccessToken() {
    $currentAzureContext = Get-AzContext
    $azProfile = [Microsoft.Azure.Commands.Common.Authentication.Abstractions.AzureRmProfileProvider]::Instance.Profile
    $profileClient = New-Object Microsoft.Azure.Commands.ResourceManager.Common.RMProfileClient($azProfile)
    $token = $profileClient.AcquireAccessToken($currentAzureContext.Tenant.TenantId)
    $token.AccessToken
}

# create authentication header
$token = get-azCachedAccessToken 
$authHeader = @{
    'Authorization' = "Bearer $token"
    'Content-Type'  = "application/json"
}

# create table
Write-Output "Creating new table '$tableName'.."
$uri = "https://management.azure.com$($logAnalyticsworkspaceId)/tables/$($tableName)?api-version=2021-12-01-preview"
$jsonBody = [String](Get-Content -Path .\table.json)
Invoke-RestMethod -Method Put -Uri $uri -Headers $authHeader -Body $jsonBody 
```

The result is a `custom Log Analyitcs workspace Table`:

![resources table](/_images/2023-06-23-log-analytics-ingest-resourcegraphdata-resourcestable.png)

### Data collection endpoint

<img src="/_images/2023-06-23-log-analytics-ingest-resourcegraphdata-datacollectionendpoint.png" width="50">

To have an endpoint to send our data to, we have to setup a `Data collection endpoint`.

To setup the `Data collection endpoint`, I use this ARM template:

`dataCollectionEndpoint.json`

``` json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "dataCollectionEndpointName": {
            "type": "string",
            "metadata": {
                "description": "Specifies the name of the Data Collection Endpoint to create."
            }
        },
        "location": {
            "type": "string",
            "defaultValue": "westeurope",
            "allowedValues": [
                "westus2",
                "eastus2",
                "eastus2euap",
                "westeurope"
            ],
            "metadata": {
                "description": "Specifies the location in which to create the Data Collection Endpoint."
            }
        }
    },
    "resources": [
        {
            "type": "Microsoft.Insights/dataCollectionEndpoints",
            "name": "[parameters('dataCollectionEndpointName')]",
            "location": "[parameters('location')]",
            "apiVersion": "2021-04-01",
            "properties": {
                "networkAcls": {
                "publicNetworkAccess": "Enabled"
                }
            }
        }
    ],
    "outputs": {
        "dataCollectionEndpointId": {
            "type": "string",
            "value": "[resourceId('Microsoft.Insights/dataCollectionEndpoints', parameters('dataCollectionEndpointName'))]"
        }
    }
}
```

and use the following PowerShell script to deploy:

``` PowerShell
# variables
$location = "<your deployment location>"
$resourceGroup = "<your resource group>"
$dataCollectionEndpointName = "<your data collection endpoint name>"

# deploy dataCollectionEndpoint 
Write-Output "Deploying Data Collection Endpoint '$dataCollectionEndpointName'.."
New-AzResourceGroupDeployment -ResourceGroupName $resourceGroup -Name "D_dataCollectionEndpoint" -TemplateFile .\dataCollectionEndpoint.json `
    -dataCollectionEndpointName $dataCollectionEndpointName `
    -location $location 
```

### Data collection rule

<img src="/_images/2023-06-23-log-analytics-ingest-resourcegraphdata-datacollectionrule.png" width="50">

This is where the magic happens, with the Data collection rule, we can setup:

- **dataCollectionEndpointId**: the `resourceId` of the `Data collection endpoint` created earlier
- **streamDeclarations**: This unique handle describes a set of data sources that will be transformed and schematized as one type. Each data source requires one or more streams, and one stream can be used by multiple data sources. All data sources in a stream share a common schema. Use multiple streams, for example, when you want to send a particular data source to multiple tables in the same `Log Analytics workspace`. 
Define a schema for the source of data. This is the schema of the `Resources`  query that we fetched from `Azure Resource Graph`
- **destinations**: This section contains a declaration of all the destinations where the data will be sent. Only `Log Analytics workspace` is currently supported as a destination. Each `Log Analytics workspace` destination requires the full workspace `resourceId` and a friendly name that will be used elsewhere in the `DCR` to refer to this workspace.
- **dataFlows**: This section ties the other sections together. It defines the following properties for each stream declared in the streamDeclarations section:
  - **streams**: This unique handle describes a set of data sources that will be transformed and schematized as one type. Each data source requires one or more streams, and one stream can be used by multiple data sources. All data sources in a stream share a common schema. Use multiple streams, for example, when you want to send a particular data source to multiple tables in the same `Log Analytics workspace`.
  - **destinations**: is the transformation applied to the data that was sent in the input shape described in the streamDeclarations section to the shape of the target table.
  - **transformKql**: Is the transformation applied to the data that was sent in the input shape described in the streamDeclarations section to the shape of the target table. In this usecase we will extend the table with the allowed fields in the `custom Log Analytics workspace table`.
  - **outputStream**: Describes which table in the workspace specified under the destination property the data will be ingested into. The value of outputStream has the Microsoft-[tableName] shape when data is being ingested into a standard `Log Analytics table`, or Custom-[tableName] when ingesting data into a custom-created table. Only one destination is allowed per stream.

To deploy the Data Collection Rule, I use this ARM template:


`dataCollectionRules.json`
``` json 
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "dataCollectionRuleName": {
            "type": "string",
            "metadata": {
                "description": "Specifies the name of the Data Collection Rule to create."
            }
        },
        "location": {
            "type": "string",
            "defaultValue": "westeurope",
            "allowedValues": [
                "westus2",
                "eastus2",
                "eastus2euap",
                "westeurope"
            ],
            "metadata": {
                "description": "Specifies the location in which to create the Data Collection Rule."
            }
        },
        "workspaceResourceId": {
            "type": "string",
            "metadata": {
                "description": "Specifies the Azure resource ID of the Log Analytics workspace to use."
            }
        },
        "endpointResourceId": {
            "type": "string",
            "metadata": {
                "description": "Specifies the Azure resource ID of the Data Collection Endpoint to use."
            }
        }
    },
    "resources": [
        {
            "type": "Microsoft.Insights/dataCollectionRules",
            "name": "[parameters('dataCollectionRuleName')]",
            "location": "[parameters('location')]",
            "apiVersion": "2021-09-01-preview",
            "properties": {
                "dataCollectionEndpointId": "[parameters('endpointResourceId')]",
                "streamDeclarations": {
                    "Custom-Resources_CL": {
                        "columns": [
                            {
                                "name": "id",
                                "type": "string"
                            },
                            {
                                "name": "name",
                                "type": "string"
                            },
                            {
                                "name": "type",
                                "type": "string"
                            },
                            {
                                "name": "tenantId",
                                "type": "string"
                            },
                            {
                                "name": "kind",
                                "type": "dynamic"
                            },
                            {
                                "name": "location",
                                "type": "dynamic"
                            },
                            {
                                "name": "resourceGroup",
                                "type": "dynamic"
                            },
                            {
                                "name": "subscriptionId",
                                "type": "string"
                            },
                            {
                                "name": "managedBy",
                                "type": "dynamic"
                            },
                            {
                                "name": "sku",
                                "type": "dynamic"
                            },
                            {
                                "name": "plan",
                                "type": "dynamic"
                            },
                            {
                                "name": "properties",
                                "type": "dynamic"
                            },
                            {
                                "name": "tags",
                                "type": "dynamic"
                            },
                            {
                                "name": "identity",
                                "type": "dynamic"
                            },
                            {
                                "name": "zones",
                                "type": "dynamic"
                            },
                            {
                                "name": "extendedLocation",
                                "type": "dynamic"
                            }
                        ]
                    }
                },
                "destinations": {
                    "logAnalytics": [
                        {
                            "workspaceResourceId": "[parameters('workspaceResourceId')]",
                            "name": "myworkspace"
                        }
                    ]
                },
                "dataFlows": [
                    {
                        "streams": [
                            "Custom-Resources_CL"
                        ],
                        "destinations": [
                            "myworkspace"
                        ],
                        "transformKql": "source 
                        | extend TimeGenerated=now() 
                        | extend resourceType=[\"type\"]
                        | extend resourceId=id
                        | extend resourceTenantId=tenantId
                        | extend resourceKind=[\"kind\"]",
                        "outputStream": "Custom-Resources_CL"
                    }
                ]
            }
        }
    ],
    "outputs": {
        "dataCollectionRuleId": {
            "type": "string",
            "value": "[resourceId('Microsoft.Insights/dataCollectionRules', parameters('dataCollectionRuleName'))]"
        }
    }
}
```

To deploy the Data Collection rule, you can use this PowerShell script:

``` PowerShell
# variables
$location = "<your deployment location>"
$resourceGroup = "<your resource group>"
$dataCollectionRuleName = "<data collection rule name>"
$logAnalyticsWorkspaceResourceId = "<your log analytics workspace resource id>"
$dataCollectionEndpointResourceId = "<data collection endpoint resource id>"

# deploy dataCollectionRule
Write-Output "Deploying Data Collection Rule '$dataCollectionRuleName'.."
New-AzResourceGroupDeployment -ResourceGroupName $resourceGroup -Name "D_dataCollectionRule" -TemplateFile .\dataCollectionRules.json `
    -dataCollectionRuleName $dataCollectionRuleName `
    -location $location `
    -workspaceResourceId $logAnalyticsWorkspaceResourceId `
    -endpointResourceId $dataCollectionEndpointResourceId
``` 

### Logic App

<img src="/_images/2023-06-23-log-analytics-ingest-resourcegraphdata-logicapp.png" width="50">


Now everything is prepared for ingestion, we can setup the `Logic App`. The `Logic App` will use a `User Assigned Managed Identity` that is linked with the `Logic App`. Form more details on the setup look at the article [Authenticate access to Azure resources with managed identities in Azure Logic Apps]

 [Authenticate access to Azure resources with managed identities in Azure Logic Apps]:https://learn.microsoft.com/en-us/azure/logic-apps/create-managed-service-identity?tabs=standard

The `User Assigned Managed Identity` will allow us to authenticate securely to `Azure Resource Graph` and ingest the data into the `Custom Log Analytics workspace table`.

We need the following `Role Based Access Control (RBAC)` role assignments for the `User Assigned Managed Identity`:

- `Reader` role on the `subscription`: To read out the resources of the subscription
- `Monitoring Metrics Publisher` role on the `Data collection rule`: permissions to post data to the `Data collection rule` and `Data collection endpoint`

> **NOTE:** It can take some time to (up to 30 minutes) get the actual permissions to the Data collection rule! 

Here's the Logic App Flow:

<img src="/_images/2023-06-23-log-analytics-ingest-resourcegraphdata-logicappdesigner.png">

#### Logic App Design

- A few **variables** are set
 
  - **DceURI**: the Uri of the Data collection rule (can be found on the Data collection rule Essentials view in the portal)
 
  - **DcrImmutableId**: The immutableId property of the DCR object (can be found on the Data collection rule Essentials `JSON view` in the portal)
  
  - **CustomTableName**: The name of the custom table (`Resource_CL`)
  
- **HTTP - Get Resource Graph Resources**
  
  Using the `Azure Resource Graph API` query the `Resources`. The `User Assigned Managed Identity` is used for this action.
  
  <img src="/_images/2023-06-23-log-analytics-ingest-resourcegraphdata-logicappdesigner_getresourcegraphresources.png">

- **HTTP - Post - Resources to Data Collection Endpoint**
  
  Concatinate the variables to a usable endpoint for ingestion: *concat(variables('DceURI'),'/dataCollectionRules/',variables('DcrImmutableId'),'/streams/Custom-',variables('CustomTableName'),'?api-version=2021-11-01-preview')*

  Ingest the `data` field of the previously fetched `Azure Resource Graph` query the `Resources` data.

  The `User Assigned Managed Identity` is used for this action.

  <img src="/_images/2023-06-23-log-analytics-ingest-resourcegraphdata-logicappdesigner_postresourcesdce.png">

## The Result

The data from `Azure Resource Graph` will now be ingested into the `Resources_CL` custom table every 15 minutes.

You can query the latest records in the `Resource_CL` table using the following query:

``` Ruby  
Resources_CL | summarize arg_max(TimeGenerated,*) by resourceId
```

The Result looks like this:

  <img src="/_images/2023-06-23-log-analytics-ingest-resourcegraphdata-lawresult.png">

You can nog start to join the table data with other data in the platform log default tables.