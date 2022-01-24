# Unattached Disks

## Purpose

Find unattached disks.

## The Query

```KQL
Resources
| where type == "microsoft.compute/disks"
| where tostring(properties.diskState) != "Attached"
| extend DiskState = tostring(properties.diskState) 
| project name, id, location, resourceGroup, subscriptionId,DiskState
```
