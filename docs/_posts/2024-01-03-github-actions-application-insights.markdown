---
layout: post
title: 'Github Actions monitoring with Application Insights'
date: 2024-01-03
logo: 'github'
description: This post explains how to monitor Github Actions with Application Insights by submitting request and annotation data to Application Insights 
image: /images/2024-01-03-github-actions-application-insights-intro.png
comments: true
---

- [Introduction](#introduction)
- [Ingest Application Insights Request Telemetry data](#ingest-application-insights-request-telemetry-data)
  - [Github Action](#github-action)
    - [PowerShell script](#powershell-script)
  - [Result](#result)
- [Ingest Application Insights Release Annotation](#ingest-application-insights-release-annotation)
  - [Github Action](#github-action-1)
    - [PowerShell script](#powershell-script-1)
  - [Result](#result-1)


## Introduction

I love working with Github Actions for Continuous Integration and Deployment.

By default you get emails regarding failed workflows for Github monitoring.

To get more insights in the performance and success of workflows, you can leverage Application Insights to log performance, failures and release annotations.  

![introduction](/images/2024-01-03-github-actions-application-insights-intro.png)

To achieve this, I use:
- The [Azure SDK for .NET ] to ingest Request Telemetry data
- The Application insights Rest API together with AZ CLI to ingest [release annotations].

[Azure SDK for .NET ]:https://learn.microsoft.com/en-us/dotnet/api/microsoft.applicationinsights.datacontracts.requesttelemetry?view=azure-dotnet

[release annotations]:https://learn.microsoft.com/en-us/azure/azure-monitor/app/release-and-work-item-insights?tabs=release-annotations#create-release-annotations-with-the-azure-cli


## Ingest Application Insights Request Telemetry data

In this example I'm logging a successful request *(ResponseCode = 200)* to Application Insights. 


The PowerShell script logs the `duration of the request`, `success` and some custom `properties` (ActionRunURL, MyTestProperty1, MyTestProperty2) 

### Github Action

The Github Action consist of the following:

- Workflow trigger:
  - [workflow dispatch] trigger to trigger the workflow manually.
- Environment variable:
  - `APPLICATIONINSIGHTS_INSTRUMENTATIONKEY`: Containing the Application Insights Instrumentation keye
- Jobs:
  - `application-insights-log-request`
    - Steps:
      - `set-startdatetime-var`: gets the current time and sets it to an environment variable
      - `login-azure`: As explained in [Use GitHub Actions to connect to Azure]
      - `application-insights-log-request`: runs a PowerShell script to log the request to Application Insights

[workflow dispatch]:https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch

[Use GitHub Actions to connect to Azure]:https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows

Here's the full content of the Github Action:

``` yaml
name: Log - Application Insights - Request

on:
  workflow_dispatch

env:
  APPLICATIONINSIGHTS_INSTRUMENTATIONKEY: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

jobs:
  application-insights-log-request:
    runs-on: ubuntu-latest
    steps:
      - name: set-startdatetime-var
        run: |
          START_DATETIME=$(date '+%d/%m/%Y %H:%M:%S')
          echo $START_DATETIME
          echo "START_DATETIME=$START_DATETIME" >> $GITHUB_ENV
      - name: login-azure
        uses: azure/login@v1
        with:
          creds: '{"clientId":"${{ secrets.AZURE_CLIENT_ID }}","clientSecret":"${{ secrets.AZURE_SECRET }}","subscriptionId": "${{ secrets.AZURE_SUBSCRIPTION_ID }}","tenantId":"${{ secrets.AZURE_TENANT_ID }}"}'
          enable-AzPSSession: true
      - name: application-insights-log-request
        shell: pwsh
        run: |

          # set variables
          $InstrumentationKey = $env:APPLICATIONINSIGHTS_INSTRUMENTATIONKEY
          $Name = $env:GITHUB_WORKFLOW
          $Started = $env:START_DATETIME
          $Success = $true # set to false for failed
          $ResponseCode = 200 # set to 500 for failed
          $startedDateTime = [datetime]::ParseExact($Started, "dd/MM/yyyy HH:mm:ss", $Null)

          [System.TimeSpan]$Duration = New-TimeSpan -Start $startedDateTime -End (Get-Date)
          $StartedDateTimeOffset = [System.DateTimeOffset]$startedDateTime

          # setup telemetry client
          $telemetryClient = New-Object -TypeName Microsoft.ApplicationInsights.TelemetryClient
          $telemetryClient.InstrumentationKey = $InstrumentationKey

          # initialize context information
          $TelemetryClient.Context.User.Id = "$($env:GITHUB_ACTOR)"
          $telemetryClient.Context.Operation.Name = $Name

          # set telemetry context
          $TelemetryClient.Context.Session.Id = "$($env:GITHUB_REPOSITORY)/$($env:GITHUB_WORKFLOW)/$($env:GITHUB_RUN_ID)/$($env:GITHUB_JOB)"
          $telemetryClient.Context.Operation.Id = "$($env:GITHUB_REPOSITORY)/$($env:GITHUB_WORKFLOW)/$($env:GITHUB_RUN_ID)/$($env:GITHUB_JOB)"

          Write-Host "SessionId: $($TelemetryClient.Context.Session.Id)"
          Write-Host "User Id: $($TelemetryClient.Context.User.Id)"
          Write-Host "Operation Id: $($TelemetryClient.Context.Operation.Id)"
          Write-Host "Operation Name: $($TelemetryClient.Context.Operation.Name)"

          # setup request telemetry information
          $request = New-Object -TypeName Microsoft.ApplicationInsights.DataContracts.RequestTelemetry
          $request.Name = $Name
          $request.StartTime = $StartedDateTimeOffset 
          $request.Duration = $Duration
          $request.Success = $Success
          $request.ResponseCode = $ResponseCode  

          Write-Host "StartTime: $StartedDateTimeOffset"
          Write-Host "Duration: $Duration"
          Write-Host "Success: $Success"
          Write-Host "ResponseCode: $ResponseCode"

          # set actionurl
          $ActionRunURL = "$($env:GITHUB_SERVER_URL)/$($env:GITHUB_REPOSITORY)/actions/runs/$($env:GITHUB_RUN_ID)"
          Write-Host "ActionRunURL : $ActionRunURL"

          # set properties 
          [HashTable]$Properties = @{}
          $properties["ActionRunURL"] = "$ActionRunURL"
          $properties["MyTestProperty1"] = "Test 1"
          $properties["MyTestProperty2"] = "Test 2"

          # copy properties to request
          $properties.Keys | ForEach-Object { $request.Properties[$_] = $Properties[$_] }

          # track the request
          $telemetryClient.TrackRequest($request)

          # flush
          $telemetryClient.Flush()

```

#### PowerShell script

Here's the content of the inline PowerShell script:

``` powershell
# set variables
$InstrumentationKey = $env:APPLICATIONINSIGHTS_INSTRUMENTATIONKEY
$Name = $env:GITHUB_WORKFLOW
$Started = $env:START_DATETIME
$Success = $true # set to false for failed
$ResponseCode = 200 # set to 500 for failed
$startedDateTime = [datetime]::ParseExact($Started, "dd/MM/yyyy HH:mm:ss", $Null)

[System.TimeSpan]$Duration = New-TimeSpan -Start $startedDateTime -End (Get-Date)
$StartedDateTimeOffset = [System.DateTimeOffset]$startedDateTime

# setup telemetry client
$telemetryClient = New-Object -TypeName Microsoft.ApplicationInsights.TelemetryClient
$telemetryClient.InstrumentationKey = $InstrumentationKey

# initialize context information
$TelemetryClient.Context.User.Id = "$($env:GITHUB_ACTOR)"
$telemetryClient.Context.Operation.Name = $Name

# set telemetry context
$TelemetryClient.Context.Session.Id = "$($env:GITHUB_REPOSITORY)/$($env:GITHUB_WORKFLOW)/$($env:GITHUB_RUN_ID)/$($env:GITHUB_JOB)"
$telemetryClient.Context.Operation.Id = "$($env:GITHUB_REPOSITORY)/$($env:GITHUB_WORKFLOW)/$($env:GITHUB_RUN_ID)/$($env:GITHUB_JOB)"

Write-Host "SessionId: $($TelemetryClient.Context.Session.Id)"
Write-Host "User Id: $($TelemetryClient.Context.User.Id)"
Write-Host "Operation Id: $($TelemetryClient.Context.Operation.Id)"
Write-Host "Operation Name: $($TelemetryClient.Context.Operation.Name)"

# setup request telemetry information
$request = New-Object -TypeName Microsoft.ApplicationInsights.DataContracts.RequestTelemetry
$request.Name = $Name
$request.StartTime = $StartedDateTimeOffset 
$request.Duration = $Duration
$request.Success = $Success
$request.ResponseCode = $ResponseCode  

Write-Host "StartTime: $StartedDateTimeOffset"
Write-Host "Duration: $Duration"
Write-Host "Success: $Success"
Write-Host "ResponseCode: $ResponseCode"

# set actionurl
$ActionRunURL = "$($env:GITHUB_SERVER_URL)/$($env:GITHUB_REPOSITORY)/actions/runs/$($env:GITHUB_RUN_ID)"
Write-Host "ActionRunURL : $ActionRunURL"

# set properties 
[HashTable]$Properties = @{}
$properties["ActionRunURL"] = "$ActionRunURL"
$properties["MyTestProperty1"] = "Test 1"
$properties["MyTestProperty2"] = "Test 2"

# copy properties to request
$properties.Keys | ForEach-Object { $request.Properties[$_] = $Properties[$_] }

# track the request
$telemetryClient.TrackRequest($request)

# flush
$telemetryClient.Flush()
```

### Result

When the workflow ran, the following result is shown: 

![Request Workflow Run](/images/2024-01-03-github-actions-application-insights-request-workflow-main.png)


![Request Workflow Run](/images/2024-01-03-github-actions-application-insights-request-workflow-run.png)

In the Application Insights instance, the following appears:

![Request Operation Workflow Details](/images/2024-01-03-github-actions-application-insights-request-operation.png)

When clicking on the request, the following details are shown:

![Request Operation Application Insights](/images/2024-01-03-github-actions-application-insights-request-operation-details.png)

## Ingest Application Insights Release Annotation

In this example I'm logging a `release annotation` for a new release with some `custom properties` from the default Github environment variables.

The trigger of the request is a 


### Github Action

The Github Action consist of the following:

- Workflow trigger:
  - [pull request] on the `main` branch of the repository.
- Environment variable:
  - `APPLICATION_INSIGHTS_RESOURCE_ID`: Containing the Application Insights Resource Id
- Jobs:
  - `create-applicationinsightsannotation`
    - Steps:
      - `set-startdatetime-var`: gets the current time and sets it to an environment variable
      - `login-azure`: As explained in [Use GitHub Actions to connect to Azure]
      - `create-applicationinsightsannotation`: runs a PowerShell script to ingest an annotation in application insights

[pull request]:https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request

[Use GitHub Actions to connect to Azure]:https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows

Here's the full content of the Github Action:

``` yaml
name: Log - Application Insights - Annotation

on:
  pull_request:
    branches:
    - main

env:
  APPLICATION_INSIGHTS_RESOURCE_ID: "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-weu-prd-githubactionsapplicationinsights/providers/microsoft.insights/components/appi-weu-prd-githubactionsapplicationinsights"

jobs:
  create-applicationinsightsannotation:
    runs-on: ubuntu-latest
    steps:
      - name: login-azure
        uses: azure/login@v1
        with:
          creds: '{"clientId":"${{ secrets.AZURE_CLIENT_ID }}","clientSecret":"${{ secrets.AZURE_SECRET }}","subscriptionId": "${{ secrets.AZURE_SUBSCRIPTION_ID }}","tenantId":"${{ secrets.AZURE_TENANT_ID }}"}'
          enable-AzPSSession: true
      - name: set-startdatetime-var
        run: |
          START_DATETIME=$(date '+%d/%m/%Y %H:%M:%S')
          echo $START_DATETIME
          echo "START_DATETIME=$START_DATETIME" >> $GITHUB_ENV
      - name: create-applicationinsightsannotation
        id: create-applicationinsightsannotation
        uses: azure/powershell@v1
        with:
          inlineScript: |

            # set annotation properties
            $annotationProperties = @{"ReleaseDescription" = "$env:GITHUB_REF"; "TriggerBy" = "$env:GITHUB_ACTOR" }
            $annotationPropertiesJson = ConvertTo-Json $annotationProperties -Compress

            # build annotation
            $annotation = @{
                Id             = [GUID]::NewGuid();
                AnnotationName = "$($env:GITHUB_REF_NAME)";
                EventTime      = (Get-Date).ToUniversalTime().GetDateTimeFormats("s")[0];
                Category       = "Deployment"; 
                Properties     = $annotationPropertiesJson
            }

            # set body 
            $body = (ConvertTo-Json $annotation -Compress) -replace '(\\+)"', '$1$1"' -replace "`"", "`"`""

            # submit annotation
            az rest --method put --uri "$($env:APPLICATION_INSIGHTS_RESOURCE_ID)/Annotations?api-version=2015-05-01" --headers "Content-Type=application/json" --body "$($body) "

          azPSVersion: "11.1.0"
          errorActionPreference: stop
```

#### PowerShell script

``` powershell
# set annotation properties
$annotationProperties = @{"ReleaseDescription" = "$env:GITHUB_REF"; "TriggerBy" = "$env:GITHUB_ACTOR" }
$annotationPropertiesJson = ConvertTo-Json $annotationProperties -Compress

# build annotation
$annotation = @{
    Id             = [GUID]::NewGuid();
    AnnotationName = "$($env:GITHUB_REF_NAME)";
    EventTime      = (Get-Date).ToUniversalTime().GetDateTimeFormats("s")[0];
    Category       = "Deployment"; 
    Properties     = $annotationPropertiesJson
}

# set body 
$body = (ConvertTo-Json $annotation -Compress) -replace '(\\+)"', '$1$1"' -replace "`"", "`"`""

# submit annotation
az rest --method put --uri "$($env:APPLICATION_INSIGHTS_RESOURCE_ID)/Annotations?api-version=2015-05-01" --headers "Content-Type=application/json" --body "$($body) "
```

### Result

When the workflow ran, the following result is shown:

![Annotation Result Main](/images/2024-01-03-github-actions-application-insights-annotation-workflow-main.png)

![Annotation Result Run](/images/2024-01-03-github-actions-application-insights-annotation-workflow-run.png)

As result, the release annotation is shown in the graph with some custom properties:

![Annotation Result Application Insights](/images/2024-01-03-github-actions-application-insights-annotation-result.png)
