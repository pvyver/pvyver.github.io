---
layout: post
title: 'Kusto Time Series Analysis for Azure Resources - Workbook'
date:   2023-06-12 
logo: 'fa fa-table-columns'
comments: true
---

## Introduction

As explained in previous [blog post], you can use the [Kusto] Query Language (KQL) for [Anomaly detection and forecasting]. 

[Anomaly detection and forecasting]:https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/anomaly-detection
[Kusto]:https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/
[blog post]:https://blog.philipvandevyver.com/2023/05/31/kusto-time-series-analysis/

This is great for ad-hoc querying the logs, and visualizing the data with the [metrics explorer]. However, having a tool ready to explore the metrics is much better.

[metrics explorer]:https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/metrics-getting-started

[Azure Monitor Workbooks] are the perfect tool for that.

[Azure Monitor Workbooks]:https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-getting-started

For your convenience, I've created an example workbook.

## Workbook

### Parameters

![Parameters](/_images/2023-06-12-kusto-time-series-analysis-workbook-parameters.png)

`Timerange`: The TimeRange  

`Subscription`: The Subscription 

`Workspace`: The Log Analytics Workspace 

`ResourceType`: The Resource Type to filter out

`Resource`: The Resource Selected

`Metric`: The Metric

`Aggregation`: The type of aggregation (Avg, Min, Max, Pct5, Pct10, Pct50, Pct90, Pct95)


### Forecast 

This is done using the [series_decompose_forecast()] function.

[series_decompose_forecast()]:https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/series-decompose-forecastfunction

![Forecast](/_images/2023-06-12-kusto-time-series-analysis-workbook-forecast.png)

### Trendline 

This is done using the [series_fit_line()] function.

[series_fit_line()]:https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/series-fit-linefunction

![Trendline](/_images/2023-06-12-kusto-time-series-analysis-workbook-trendline.png)

### Anomalies

This is done using the [series_decompose_anomalies()] function.

[series_decompose_anomalies()]:https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/series-decompose-anomaliesfunction

I also added a table that extracts the `outliers` for the anomalies using the [mv-expand] operator on the timeseries.

[mv-expand]:https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/mvexpandoperator

![Anomalies](/_images/2023-06-12-kusto-time-series-analysis-workbook-anomalies.png)

## Download

Here's the [ARM Template]   the [Gallery Template].

[ARM Template]:https://raw.githubusercontent.com/pvyver/workbooks/main/Azure%20Services/Time%20Series%20Analysis/armTemplate/template.json

[Gallery Template]:https://raw.githubusercontent.com/pvyver/workbooks/main/Azure%20Services/Time%20Series%20Analysis/galleryTemplate/template.json

