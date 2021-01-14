# AKS Pod Status

## Purpose

Get all the pods (excluding system pods) info like status, number of restarts, name, and more.

```KQL
let podTable = KubePodInventory
| where isnotempty(ClusterName)
| where isnotempty(Namespace)
| where isnotempty(Computer)
| where Namespace != "monitoring" and Namespace != "aad-pod-id" and Namespace != "kube-system" 
| summarize arg_max(TimeGenerated, *) by Name
| project TimeGenerated, PodName=Name, Node=Computer, PodType=ControllerKind, PodStatus, ContainerStatus, ContainerStatusReason, PodRestart=PodRestartCount, PodCreationTimeStamp, ClusterName, Namespace;
let clusterTable = KubeNodeInventory
| summarize arg_max(TimeGenerated, *) by ClusterId
| extend ClusterRegion = tostring(parse_json(Labels)[0].["topology.kubernetes.io/region"])
| project ClusterId, ClusterName, ClusterRegion;
podTable
| join kind=inner clusterTable on ClusterName
| project PodName, ClusterName, PodStatus, ContainerStatus, ContainerStatusReason, ClusterRegion, Node, Restarts=PodRestart, Age=now()-PodCreationTimeStamp, Namespace, ClusterId, PodCreationTimeStamp, TimeGenerated
```
