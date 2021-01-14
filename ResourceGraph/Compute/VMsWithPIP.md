# VMs with Public IPs

## Purpose

Find VM's with Public IP attached to them.

## The Query

```KQL
Resources
| where type =~ 'microsoft.compute/virtualmachines'
| extend nics=array_length(properties.networkProfile.networkInterfaces)
| mv-expand nic=properties.networkProfile.networkInterfaces
| where nics == 1 or nic.properties.primary =~ 'true' or isempty(nic)
| project vmId = id, vmName = name, vmSize=tostring(properties.hardwareProfile.vmSize), nicId = tostring(nic.id)
                | join kind=leftouter (
                    Resources
                     | where type =~ 'microsoft.network/networkinterfaces'
                     | extend ipConfigsCount=array_length(properties.ipConfigurations)
                     | mv-expand ipconfig=properties.ipConfigurations
                     | where ipConfigsCount == 1 or ipconfig.properties.primary =~ 'true'
                     | project nicId = id, privateIP= tostring(ipconfig.properties.privateIPAddress), publicIpId = tostring(ipconfig.properties.publicIPAddress.id), subscriptionId)
                     on nicId
| project-away nicId1
| summarize by vmName, vmId, vmSize, nicId, privateIP, publicIpId, subscriptionId
                | join kind=leftouter (
                    Resources
                     | where type =~ 'microsoft.network/publicipaddresses'
                     | project publicIpId = id, publicIpAddress = tostring(properties.ipAddress)) on publicIpId
| project-away publicIpId1
| where  publicIpAddress != ""
| sort by publicIpAddress desc
```
