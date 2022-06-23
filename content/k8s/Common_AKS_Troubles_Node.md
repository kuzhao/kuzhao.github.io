---
title: "Common AKS Troubles - NodeNotReady"
date: 2022-06-23T14:15:00+08:00
draft: false
---
A crucial role K8s plays is to form abstraction between underlying infrastructure and the microservice framework. In real-world practice, though, things never run perfectly. In other words, chance always exists that node machines or pod applications get stuck or crashed.  
In this blog, we are going to expand on a common scenario of infrastructure failure -- **NodeNotReady**. We all know it is up to something between kubelet and api-server. Perhaps to your surprise, this topic stretches over a wide area with much uncharted water. In general, though, we can classify such problems into three kinds below.  
#### Node side problem
Kubelet, like other daemons running on the OS, is nothing more than a common program. Therefore, any troubles that undermine its processes or underlying OS memory/network are able to take it down.  
In Azure world, cloud provider incidents like VM service healing and Azure hypervisor (Azure host) updates contribute to short VM downtime and further NodeNotReady events. Furthermore, a wide VM blackout happens during a regional outage on Azure compute, unexpectedly taking down a fair percentage of AKS nodes. Check "Service Health" for VM impacting Azure outages and "Activity log" for VM/VMSS instance individual incidents.  
One step up the stack, Linux OS atop virtual hardware is also subject to change. In the case of AKS, the underlying node OS is Ubuntu 18.04 which has daily security patching enabled. You can recognize its schedule through "apt-daily-upgrade.timer" and "apt-daily-upgrade.service" in systemd. Though most of the time there is no patching or the installation is subtle enough, key system services like systemd-networkd could be bounced to apply the patch, causing eth down/up and disrupting kubelet.  

Other possible factors: kubelet OOMkilled due to low OS free memory; Filled OS disk; Too many containers which use up PIDs; Too intensive disk IO leading to IO starvation.  
#### Control plane side problem
Under the context of cloud-provider managed control plane, the performance of api-server and other parts seems to be out of concern as the managed environment is generally considered to be solid. This is also the reason why nodeNotReady has little to do with api-server on small clusters: the control plane has sufficient capacity left when handling only a limited number of nodes.  

Still, many enterprises run large scale clusters with more than 50 nodes plus multiple node pools. On this occasion, api-server is being hammered all the time from numerous kubelet instances, and the performance may degrade. As a result, contingent nodeNotReady events will emerge.  
There are signs for sure. The user will first notice unusual slowness at kubectl response. If kubectl slowness is accompanying intermittent nodeNotReady, it will be time for you to suspect a degraded api-server. As a next step, you can choose to open a support ticket for api-server performance check. But you can also do it by yourself after adding one diagnostic settings and ticking "kube-audit". Visualize the query result in log workspace with time versus http status code. Watch out for a high percentage of 5xx. 

As for the solution, so far there are not many effective methods. If you are willing to compensate control plane performance with a little extra cost, you can enable uptimeSLA which then adds two extra api-server instances. Otherwise, you will need to figure out if any heavy kubeAPI consumer apps like ArgoCD/Ambassador, and cut the replica count.  
#### Network in between
The network between kubelet and control plane works smoothly in self-hosted environments, where the master and agent nodes live on the same network backed by high-speed physical routers and switches. In the cloud environment and AKS in particular, kubelet by default communicates with api-server over public networks. Because of this, we need to put a much closer scrutiny on outbound connectivity from node VMs to the public IP of the cluster FQDN.  

A common blocker seen in my past experience is NVA, like Palo Alto, Fortinet, that sits in between vnets and the internet. If you are pointing 0.0.0.0/0 the default route to a virtual appliance, make sure that rules exist on the appliance and allow TCP 443 access to the cluster FQDN.  
If the cluster vnet has a virtual gateway which is attached to an express route circuit, there could also be implicit default route injection. You can double confirm its existence by going to the NIC of a VM in the same vnet as the cluster, and check its "Effective Routes".  
Lastly, I once mentioned the impact of SNAT exhaustion in the section "Outbound Connection Timeout From Pods", [Common AKS Troubles - Networking](https://kuzhao.github.io/k8s/common_aks_troubles_networking/). The rules there also apply to kubelet - control plane connectivity.  