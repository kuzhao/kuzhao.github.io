---
title: "Common AKS Troubles - Networking"
date: 2022-06-14T15:05:11+08:00
draft: false
---
Believe it or not, simple connectivity could have to do with a much broader range of topics that you imagine. In this blog, we are going to explore over several common problems.
#### Error Events on LoadBalancer Typed KubeService
When you expose your microservices via either altering or creating a kube service, you typically get a series of events when describing the target svc. Normally, you will see a public, non-clusterIP address at "Exteral-IP" field when kubectl get svc.  
Sometimes, you may unexpectedly not see any external IP assigned. Then it will be time to carefully examine svc events. What could go wrong then?  
The routine that runs behind the scenes is a "LoadBalancer Ensurer" module inside kube-controller-manager. When a KubeAPI call is mae to PATCH(modify) or PUT(create) a svc under type "LoadBalancer", the controller manager invokes corresponding LB ensuring methods within cloud-provider golang packages. In AKS case, azure-cloud-provider methods take over and operate on the cluster load balancer Azure resource in MC resource group.  
Exceptions at AKS LB ensurer mostly root from failed Azure resource API calls to modify frontends, rules or healthprobes at the cluster load balancer. In case it happens, parse and read carefully the failure svc events and examine activity logs of the LB resource. Usually, you will find clues within resource activities pointing to one or two LB config failures.
#### DNS Failure, Persistently or Intermittently
Long time ago I once described in detail how pod DNS works in AKS in article "[AKS NETWORKING ILLUSTRATED, PART II(CONT.)](https://kuzhao.github.io/k8s/aks_networking_ii_cont/)". As pod DNS traffic goes to coredns svc clusterIP first before getting reflected by coredns to its upstream, any bad link in this long chain can be the root cause.  
**Outgoing direction**  
Let's first examine what could happen to a DNS query during pod -> coredns -> upstream. First, the query from the client pod to coreDNS clusterIP is DNAT and redirected by iptables to one of coredns pods. In **kubenet** clusters, wrong node podCIDR route entries disrupt pod to pod traffic and cause DNS timeout.  
After the request arrives at a coredns pod, the pod then either looks up cluster internal FQDNs or forwards it upstream. There will not be any issue on the first one, while the redirection to upstream may yield timeout depending on UDP connectivity. A customized Azure route table (UDR) applied to the cluster subnet could lead to a routing blackhole. Otherwise, a firewall in the path between the cluster and upstream DNS servers may block UDP traffic.  
**Returning direction**  
Assuming the DNS answer comes from upstream, it will be relayed to the source client pod by coredns. Little problem takes place in this journey back except for a corner case: UDP DNS answer with flag "TR".  
If you search for "DNS TR" at Google, many articles show up explaining what it is. In short, it is a flag set by the upstream DNS server indicating that the answer is too long to be contained in a single UDP packet. Coredns transparently passes it back to the client, then it will be up to the client pod to handle.  
No further expansion on this topic here, but some old alpine base images are known not to be able to handle this well. Example: [musl-libc - Alpine's Greatest Weakness](https://www.linkedin.com/pulse/musl-libc-alpines-greatest-weakness-rogan-lynch/)
#### Outbound Connection Timeout From Pods
A key thing here is to clarify on the traffic path of the outbound network packet from a pod.  
When an IP packet is sent out, it first arrives at the iptables. If the destination is cluster external, the source will be "masqueraded" into the address of node's eth0. The packet then follows the same routine as a normal one from a Linux server.  
The next stop after the packet is sent from eth0 is Azure route table, which is a piece of calculating mechanism inside the hypervisor. For simplicity, we discuss here only internet destinations without any firewall in the path. The packet is then sent to the cluster load balancer before going outbound to the internet.  
As the final step, Azure LB SNATs the packet with LB frontend public address before sending it to Azure edge routers for internet.  
While the chance of a misbehaving iptables is dim, timeout in this category often takes root at LB SNAT. If you study into LB documentation, you will notice the concept of "SNAT port assignment" to each VMs in the LB backend pool which includes all cluster nodes. Things go wrong when too many pods on one node are firing too many TCP connections to the internet. The SNAT utilization goes beyond the assigned count on that node, resulting in a portion of TCP handshake SYN getting dropped at the LB.  
Symptoms of this kind can be easily noticed by looking into metrics on the cluster LB. Two metrics that will help you out: "Allocated SNAT ports" and "Used SNAT ports". Split the chart per backend IP (AKS node, in other words), then compare if any nodes experience spillover on "used" out of "allocated".