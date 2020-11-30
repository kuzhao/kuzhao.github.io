---
title: "AKS Networking, illustrated -- Part I"
date: 2020-11-30T14:37:11+08:00
draft: false
---

It has been a year since I started working on customer escalation and advisory for AKS service at Microsoft. After so many customer case studies, I do feel the need to write up a series on how this Azure product works. I hope these articles will help work in multiple roles ranging from cloud solution architecting to AKS cluster administration. 

I will talk about its **networking** as the start. And there will be three parts on this subtopic, each corresponding to one article in this series.
This will be intended for: a) people managing and monitoring their clusters; b) networking experts interested in enriching their knowledge arsenal with microservice networking; c) Azure project executors who are about to implement great Azure solutions.

### Overview
Let's first set the perimeter to properly scope what to discuss. 
- It is only about an AKS cluster, rather than an on-premise one or a cluster offered by GCE or AWS
- We will not examine K8s master logic and control plane components such as api server and controller manager.
- We will not discuss anything related to applications and workloads, even if some of them, like Istio and Nginx ingress, offer networking related services.

In this series, we will first check and name concepts and types of resources involved in any AKS networking operations or functionalities in Part I. In Part II as the next step, we will focus on the K8s concept -- a service object, which works essentially as the entry for the largest percentage of cluster traffic. We will then further categorize cluster traffic into various kinds and break them up for analysis.
Lastly, we will take the cloth and dress up as detectives before diving into some of the most bizarre yet resounding symptoms and pitfalls regarding AKS networking.

-------Part I begins here----------------
Allow me to use my ragged sketch below to name all networking related elements of an AKS cluster.
![AKS Networking Resources](/static/aks-net-azure-resource.png)

Some highlighted bullets below:
- Typical inbound traffic to the cluster will first hit the frontend address on the Azure load balancer before being distributed to AKS node VMs.
- Just like normal VMs, the subnet NSG and route table apply to all node VMs in the cluster subnet. You can introduce additional per-VM NSGs although they rarely appear in real scenarios.
- You can always introduce an application gateway as the entry point instead of the load balancer. Two use cases: 1) AGW ingress; 2) normal AGW with manually set backends of AKS load balancers. AGW always sits in a different subnet, to which the cluster subnet is reachable.
- The vnet containing the cluster subnet is of no difference from any other vnets. Feel free to config vnet peering, expressroute and VPN gateways for it. Of course, you are also free to use custom DNS servers here.

I am now moving to node VMs and checking out VM and pod networking there. Depending on the network mode you choose for your cluster, the setups differ from each other a lot. 

### Kubenet (Basic Mode)
This mode is really "basic" and applies Kubenet, one of the simple network plugins in K8s world. In this mode, each node has only one address from the cluster subnet. You need to set "podCIDR" when creating the cluster, so that pods get their IPs from this virtual address space. 

There will be a route table resource created and attached to the cluster subnet, though. This is because the podCIDR will be broken into segments and assigned to each node per Kubenet design. As a result, each node will have one entry corresponding to its network segment, with the next hop as its node IP in the route table.

### AzureCNI (Advanced Mode)
The pod IP assignment is more straightforward in this mode and you don't even need to specify the podCIDR. In contrast to Basic Mode, pods get their addresses directly from the cluster subnet. This way, the pods and nodes are in the same network address space. As a result, they are all directly addressable within the entire vnet, including peered ones and on-premises with ExpressRoute/S2SVPN in place.

One of the important features that takes place behind the scenes is: each node is pre-assigned with a number of subnet addresses equal to pod count per node in cluster config (default to 30). Therefore, the subnet address space restricts the number of nodes you can have.  It is also wise to set aside spare addresses for another node in case of a cluster upgrade.
