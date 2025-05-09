# Log Analytics Cost Analysis - Benefits and Usage

This document contains KQL queries to help analyze Log Analytics costs, benefits usage, and data ingestion patterns.

## Monitor Benefits Usage (Last 30 Days)

This query tracks the consumption of different benefit types (like M365 and P2) over the last 30 days, helping you understand your benefits utilization.

```KQL
Operation
| where TimeGenerated >= ago(30d) //Can change to 90 days if needed
| where Detail startswith "Benefit amount used" //pulls only data from operation table we need.
| parse Detail with "Benefit amount used: " BenefitUsedGB " GB" //Cleans the output some
| extend BenefitUsedGB = toreal(BenefitUsedGB)  //Makes it actual GB
| parse OperationKey with * "Benefit type used: " BenefitType //Gets the types of Benefit (M365 and P2)
| project TimeGenerated, BenefitType, BenefitUsedGB  //Puts data to clean method
| summarize TotalBenefitUsedGB=sum(BenefitUsedGB) by BenefitType  //summarizes the data for easy reading
```

## Track Defender for Servers Plan 2 License Usage

This query helps monitor your daily data ingestion against the maximum allowed data based on your Defender for Servers Plan 2 licenses. The default calculation uses 500MB per license per day as the allowance.

Key points:

- Adjust `NumberOfLicenses` to match your environment
- Default period is 30 days, can be modified to 90 days
- Monitors key security-related data types only
- Results are in GB per day

```KQL
let NumberOfLicenses = 621; //Number of licenses you have
Usage
| where IsBillable == true
| where TimeGenerated >= ago(30d) //Can change to 90 days if needed
| where DataType in ("SecurityAlert",
"SecurityBaseline",
"SecurityBaselineSummary",
"SecurityDetection",
"SecurityEvent",
"WindowsFirewall",
"ProtectionStatus",
"MDCFileIntegrityMonitoringEvents",
"WindowsEvent",
"LinuxAuditLog")
| summarize DailyIngestionGB = toreal(sum(Quantity))/ 1024  by format_datetime(TimeGenerated, 'yyyy-MM-dd')
| extend MaxDataGrantGB  = ((500*toreal(NumberOfLicenses))/1024)
| sort by TimeGenerated asc
```

## Analyze Data Ingestion by Type

This query provides a visual breakdown of data ingestion by type, helping identify which data sources contribute most to your costs.

Features:

- Shows top 10 data sources by default
- Groups smaller sources into "Other" category
- Displays daily ingestion trends
- Results visualized as a column chart

```KQL
let TopTables = materialize(
    Usage
    | where TimeGenerated > startofday(ago(31d))
    | where StartTime > startofday(ago(31d))
    | where IsBillable
    | summarize IngestedGB=sum(Quantity) / 1.0E3 by DataType
    | sort by IngestedGB desc
    | take 10 //Remove this line if you want to see all the tables.
    | project DataType);
Usage
| where TimeGenerated > startofday(ago(31d))
| where StartTime > startofday(ago(31d))
| where IsBillable
| extend Table = iff(DataType in (TopTables), DataType, "Other")
| extend Rank = iff(Table == "Other", 2, 1)
| summarize IngestedGB=sum(Quantity) / 1.0E3 by Table, bin(StartTime, 1d), Rank
| sort by Rank asc, IngestedGB desc
| project-away Rank
| project StartTime, Table, IngestedGB
| render columnchart
```