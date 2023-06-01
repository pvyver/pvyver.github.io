---
layout: post
title: 'Correlate Logs in Log Analytics with Azure Resource Graph (Tag) data'
date:   2023-03-21 
logo: 'list'
---

## Introduction

`Logs` in a `Log Analytics Workspace` are powerful for alerting, debugging and visualization purposes. 

While working with these logs, I regulary faced the issue that they (at the time of writing) cannot be correlated with information coming from `Azure Resource Graph` resource tables.
This could be helpful for alerting, chargeback on logs using tags, ...

To workaround the issue, I decided to fetch the `Azure Resource Graph` data from resources and ingest them into a Log Analytics Workspace.

## Prerequisites

- Log Analytics Workspace
- Automation Account
- User Assigned Managed Identity
    - Reader Permissions on the Subscription(s)
    - Metrics Contributor

## Approach

- Data Collection Endpoint (DCE)
- Data Collection Rule (DCR)
- Azure Automation Runbook 
    - Schedule
- Consume Log Analutics Workspace data
    - Query Logs
    - Create alert
    - Create Workbook


