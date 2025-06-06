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

## Check how much Defender for Servers Plan 2 allowance you have based on your licenses

**Purpose**: Monitor daily data ingestion against your license allowance to ensure you stay within the included data limits.

**How to use this query**:
1. Locate the `NumberOfLicenses` variable at the top
2. Update the number to match your current Defender for Servers P2 license count
3. Run the query to see:
   - Daily ingestion in GB
   - Maximum allowed data based on your licenses
4. The time range defaults to 30 days but can be adjusted
5. Results are sorted by date for easy trend analysis

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

**Purpose**: Visualize data ingestion patterns to identify high-volume data sources and optimize costs.

**How to use this query**:
1. The query uses a two-step approach:
   - First, identifies top data sources
   - Then, creates a visual representation
2. Customization options:
   - Change the time range (default: 31 days)
   - Adjust the number of top sources (default: 10)
   - Remove the `take 10` line to see all sources
3. The results show:
   - Daily ingestion per data type
   - Automatic grouping of smaller sources as "Other"
   - Visual column chart for trend analysis
4. Use this to:
   - Identify cost-driving data sources
   - Plan data retention policies
   - Make informed decisions about data collection

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

## Compare daily ingestion with daily benefit grant for Defender for Servers Plan 2 - Graph presentation

**Purpose**: Visually compare your daily Defender for Servers Plan 2 data ingestion with the daily benefit grant, making it easy to spot days where usage exceeds the included allowance.

**How to use this query**:

1. The query joins daily ingestion with the corresponding daily benefit grant.
2. Results show both values side by side for each day.
3. Use the output to:
    - Identify days where ingestion exceeds the benefit grant
    - Track trends and optimize usage
    - Visualize the comparison as a chart for quick analysis

**Customization options**:

- Adjust the time range by changing `ago(30d)` to your preferred period.
- Modify the list of `DataType` values to match your environment.
- Use the results with a line or column chart visualization for best effect.

```KQL
let benefits= Operation
| where TimeGenerated >= ago(30d) //Can change to 90 days if needed
| where Detail startswith "Benefit amount used" //pulls only data from operation table we need.
| parse Detail with "Benefit amount used: " BenefitUsedGB " GB" //Cleans the output some
| extend BenefitUsedGB = toreal(BenefitUsedGB)  //Makes it actual GB
| parse OperationKey with * "Benefit type used: " BenefitType //Gets the types of Benefit (M365 and P2)
| where BenefitType == "MicrosoftDefender" // As we only looking on the Defender for Plan 2
| project TimeGenerated, BenefitType, BenefitUsedGB;  //Puts data to clean method
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
| summarize DailyIngestionGB = toreal(sum(Quantity)) / 1024 by bin(TimeGenerated, 1d)
| join benefits on TimeGenerated
| project TimeGenerated, DailyIngestionGB, BenefitUsedGB
```

## Compare daily ingestion with daily benefit grant for Defender for Servers Plan 2 - Show if yes or no

**Purpose**: Show, for each day, whether your Defender for Servers Plan 2 data ingestion exceeded the daily benefit grant, with a simple "yes" or "no" indicator.

**How to use this query**:

1. The query compares daily ingestion with the daily benefit grant.
2. For each day, it adds a column (`BenefitOverage`) that shows "yes" if ingestion exceeded the grant, or "no" otherwise.
3. Use the results to quickly identify days where you went over your included allowance.

**Customization options**:

- Adjust the time range by changing `ago(30d)` to your preferred period.
- Modify the list of `DataType` values to match your environment.
- Use the output for compliance checks or to trigger alerts when overages occur.

```KQL
let benefits= Operation
| where TimeGenerated >= ago(30d) //Can change to 90 days if needed
| where Detail startswith "Benefit amount used" //pulls only data from operation table we need.
| parse Detail with "Benefit amount used: " BenefitUsedGB " GB" //Cleans the output some
| extend BenefitUsedGB = toreal(BenefitUsedGB)  //Makes it actual GB
| parse OperationKey with * "Benefit type used: " BenefitType //Gets the types of Benefit (M365 and P2)
| where BenefitType == "MicrosoftDefender" // As we only looking on the Defender for Plan 2
| project TimeGenerated, BenefitType, BenefitUsedGB;  //Puts data to clean method
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
| summarize DailyIngestionGB = toreal(sum(Quantity)) / 1024 by bin(TimeGenerated, 1d)
| join benefits on TimeGenerated
| project TimeGenerated, DailyIngestionGB, BenefitUsedGB
| extend BenefitOverage = iif(DailyIngestionGB  > BenefitUsedGB, "yes", "no")
```