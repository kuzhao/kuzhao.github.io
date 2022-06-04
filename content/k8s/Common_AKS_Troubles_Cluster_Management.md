---
title: "Common AKS Troubles - Cluster Management"
date: 2022-06-04T10:05:11+08:00
draft: false
---
The sequel continues. Today, the focus is on the topic of "cluster management".  
#### Cluster Management
This category goes over a wide range from simply scaling out a node to unusual cluster certificate rotation. Beware, though, that we will not bring any workload related things to the discussion, i.e. K8s deployment, svc, ingress and secret. Instead, cluster infra and config changes will be the target.  
#### Scaling Failure
AKS operators take nodepool scaling for breakfast. Still, I need to take a step further here: you need to differentiate autoscaling from manual, as the two scaling types go in fundamentally separate ways. Check further below.  
- Manual scaling  
A node pool under this scaling mode can be easily recognized by a "Manual" flag plus a scroll bar to adjust the target node count. When a manual scaling operation happens after "OK" button is clicked, an ARM request to set the nodepool to the target count is sent to Azure. AKS backend then picks it up and starts one simple thing: send a scaling request with the target count to VMSS backend.  
Perhaps you can then imagine potential types of exception -- not enough VM core quota, VM extension failure, new node NotReady, or something else. While the first possibility can be easily mitigated by increasing the quota in Azure portal, all other ones could be too complicated to handle.  
In case of extension failure, a simple understanding is that an error blocks node initialization. You should then receive a returnCode of CSE extension in the scaling failure errmsg. [This table](https://github.com/Azure/aks-engine/blob/master/pkg/engine/cse.go) may give you a first impression about which part goes wrong. When extension failure leads to node NotReady, please collect the scaling failure errmsg for the later two exception types and file a support ticket.  
- Autoscaling  
Comparatively, a nodepool under this mode is recognized through a "Auto" flag with a slider to adjust **Min and Max** count. You cannot modify the node count directly as manual mode.  
The paradigm shift from manual happens at scaling mechanism. A cluster autoscaler program, right from the [upstream project](https://github.com/kubernetes/autoscaler), is deployed inside the managed control plane, and sends scaling commands straight to VMSS backend. This is why you will not see any scaling operations in "Activities" of the cluster, but VMSS activities are still happening.  
If you think an Azure support engagement will take a long time, you may try the following steps to figure out an autoscaling problem yourself.  
**Enable Diagnostics**  
You will find an item named "cluster-autoscaler" in "Diagnostic Settings" on Azure Portal AKS resource page. Tick on it and confirm the settings change.  
**Read Autoscaler Logs**  
Then it will be time for your Log Workspace skills! Navigate to "Logs" pane on the resource page, and refer to the sample query [here](https://docs.microsoft.com/en-us/azure/aks/cluster-autoscaler#retrieve-cluster-autoscaler-logs-and-status).
#### SP Credential and Cluster Certificate Refresh Failure
While these two kinds of cluster management op do not happen frequently, you may still need to accomplish them in a limited maintenance window. Therefore, a cluster failure in the middle of progress can easily cause inoperable hours before the window draws to its end.  
Interestingly, though, the node processing part of SP Credential refresh can take similar steps with nodepool upgrade. The procedure checks if your node image is at the latest. Conceivably, the majority of clusters do not keep up with the once-every-month node image release cycle. As a result, a node image upgrade will probably occur during SP credential refresh.  
Then it makes sense to apply all blocking factors at nodepool upgrade mentioned in previous "Common AKS Troubles - CRUD" article.  
**About cluster certificate refresh**  
In this scenario, the only similarity with SP credential as well as node pool upgrade is that all nodes will be recreated. The mechanism, however, completely differs.  
Instead of rolling manner, all nodes in the pool are deleted at the same time, before new cluster CA cert and api server cert are installed in newly generated nodes. This can be verified by nodepool VMSS activities where the scale set is first deleted and then recreated.  
A positive effect of this difference, though, is that you may face way less trouble here as rolling node restarts are not going to happen. Still, when you get "operation failure", try to first tell if the failed step is at node pool/VMSS by checking if any VMSS activities have happened since the operation starts. Then clearly state your judgement in the support ticket and remember to enclose related VMSS activities if applicable.
