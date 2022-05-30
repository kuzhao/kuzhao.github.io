---
title: "Common AKS Troubles - CRUD"
date: 2022-05-30T17:15:11+08:00
draft: false
---
In these two consequtive blogs, we are going to fly over a plethora of AKS problems, highlighting where to check and poke for quick resolution.
#### AKS Resource CRUD
"CRUD" typically means all conceivable operations against an object. In the world of database, you might once hear terminologies like "atomic CRUD" on create/read/update/delete operation against data row(s).  
In the world of Azure cloud, an AKS cluster, similar with other resource types, takes CRUD requests from cloud admin. The request, whose body takes a json format, flows and routes through Azure Resource Manager (ARM), before ends up being processed at AKS resource provider (RP).  
The cluster version number hikes fast with one new minor version released every three months, while AKS service keeps only the latest three of stable versions. Every day, Azure users around the globe create, upgrade and tear down their clusters. While viewing resource json definition (Read) is generally okay, not all other operations go through well.  

**Upgrade Failure**  
This type of CRUD failure takes up the majority, maybe 4 in 5 if not less. There are two subtypes in this category:  
* Complete upgrade  
A full cluster upgrade consists of two phases: managed control plane upgrade and agentpool upgrade. The key thing here is to judge in which phase the upgrade breaks.  
It is not hard to tell. Since AKS backend orchestrates resources in node resource group, you can just go there and check resource activities in that group. If there is not a single record before upgrade failure errmsg and after the upgrade was started, then the failure takes place in the first phase. Otherwise, the upgrade merges with the second subtype on agentpools.  
In case of a phase one failure, either wait for 30min or an hour and retry the upgrade, or file a support ticket.
* Agentpool only upgrade  
Many users choose to upgrade pool separately with consideration of caution. The process here goes the same way as phase two of a full upgrade, and in a time consuming manner.  
Behind the scene, AKS backend keeps draining nodes, instructing individual VMs to stop, reimage and start one by one. The sequence of operations is reflected in VMSS resource activities. Although such operations should finish automatically, you may notice a terminating failure record after receiving the "upgrade failed" message.  
Typical VM operation failures root from CSE error, VM stop/start timeout/failure and node draining failure. Examine closely the VMSS failed activity and try to determine the reason and/or file a support ticket.

**Creation Failure**  
When this happens, there should be an error message, though perhaps in an ugly format. If in Azure portal, it pops out in topright corner and is viewable in both resource group "deployment" and resource "Activities"; if in Azure CLI, it is output to stderr of the terminal.  
In the error message, look for clues pointing to Azure user/servicePrincipal permission, Azure Policy forbiddance, and affiliated resource creation failure, i.e. VMSS, public IP and Azure load balancer. 
If you are to raise a support case, please do enclose the output in support ticket statement.  
**Deletion Failure**  
With continuous platform and product evolution in AKS service, we rarely encounter such kind of failure in the recent two years. If it does happen, the error message is most likely pointing at missing permissions or unexpected resource removal exceptions.  
After ruling out permission and Azure policy misconfiguration, try to narrow down on blocked resource removal. You can then attempt manual resource removals before reaching out to Azure support.