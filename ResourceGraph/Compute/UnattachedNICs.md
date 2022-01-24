# Unattached NICs

## Purpose

Find Unattached NICs.

## The Query

```KQL
resources
| where type == "microsoft.network/networkinterfaces"
| extend AttachedVM = properties.virtualMachine
| where AttachedVM == ""
| project name, id, location, resourceGroup, subscriptionId, AttachedVM
```
