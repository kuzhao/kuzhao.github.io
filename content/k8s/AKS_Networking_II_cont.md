---
title: "AKS Networking Illustrated, Part II(cont.)"
date: 2021-01-20T13:00:11+08:00
draft: false
---
*The Series on AKS Networking --*  
[AKS Networking Illustrated, Part I](https://kuzhao.github.io/k8s/aks_networking_i/)  
[AKS Networking Illustrated, Part II](https://kuzhao.github.io/k8s/aks_networking_ii/)  
[AKS Networking Illustrated, Part II(cont.)](https://kuzhao.github.io/k8s/aks_networking_ii_cont/)  
[AKS Networking Illustrated, Part III](https://kuzhao.github.io/k8s/aks_networking_iii/)  

Wrapping up K8s services and LB in AKS, let's now further examine yet another type of traffic -- cluster network services to pods. Though not in large volume, it is crucial for wellbeing and performance of an AKS cluster. 

#### DNS
Being a cluster internal service, coreDNS -- the DNS software image used in AKS -- has a clear traffic path:  
Pod -> coreDNS ClusterIP -> a coreDNS pod -> Upstream DNS servers  
In the context of Azure, the default upstream which is used to resolve internet FQDNs is the DNS server configured for the virtual network of the cluster subnet. Zooming it in, a coreDNS behaves this way to reach out for an internet FQDN:  
coreDNS pod -> serverAddr in resolv.conf of the hosting node -> vnet DNS server  
Eventually, it would be either Azure offered DNS service @168.63.129.16 or a custom server configured by yourself or your Azure networking admin.

This concludes all DNS related to the below map:  
![AKS DNS](/img/aks_dns.png)

#### Tunnel
In a nutshell, the cluster tunnel serves as the passage through which the control plane reaches out to the node it wants. Note that this concept is Azure specific and does not apply to other K8s implementations.  
Given such little info printed on the AKS official doc, it would be worthwhile to talk more about how traffic works out here. Refer to the map below:  
![AKS tunnel](/img/aks_tunnel.png)  
Note that apart from handling requests on logs and exec, it is also responsible for passing all instructions the control plane pushes to the node pool. Considerably, once it's down you can no longer deploy new pods as the control plane can no longer reach kubelet on nodes.

#### Container Insight
We are explicitly talking about "Insights" and "Logs" in the cluster blade of Azure Portal. And the ops data comes from agents that are automatically deployed in kube-system by AKS service.  
A direct way of thinking on this subtopic is to treat those logging and telemetry agents as plain apps but in kube-system. There are two parts to the relevant cluster workloads:
- omsagent-rs (ReplicaSet)
- omsagent (DaemonSet)

Examined from cluster operation perspective, they communicate with external web services just like other web app workloads. In addition, you will find through packet dump that they mostly talk with these Azure service FQDNs:
- [regionName].monitoring.azure.com
- [uuid].ods.opinsights.azure.com
- dc.service.visualstudio.com
- api-server

Therefore, better keep those FQDNs reachable if you have a firewall service standing in the way of cluster egress traffic (usually by applying extra route table entries to the cluster subnet).