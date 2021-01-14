# Firewall Data Processed

## Purpose

Get a Graph with Azure Firewall Data Processed and display in a chart.

## The Query

```KQL
AzureMetrics 
| where Resource == "<FireWall Name>"
| where MetricName == "DataProcessed"
| project MetricName, Total, Average, UnitName, TimeGenerated
| render timechart
```
