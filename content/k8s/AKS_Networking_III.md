---
title: "AKS Networking illustrated, Part III"
date: 2021-08-27T11:15:11+08:00
draft: false
---
In this blog we will review networking troubles from the real world -- on decent, serious production clusters in my customer support career. Be prepared!

#### What's wrong with the api-server?
As part of the brain of the cluster, the api server is an essential piece that plays a part in various scenarios. I am just wondering what's gonna happen if it becomes out of reach.  
There are two aspects to this:  
- For AKS nodes, one will go "NotReady" if losing contact.
- For pod apps, the results vary from log exception to crashloop.

One day my case bell rang with a customer complaining about part of nodes NotReady. The following loglines appear in syslog:
```
Error updating node status, will retry: error getting node "aks-agentpool01-27254751-vmss000030": unexpected error when reading response body. Please try again. Original error: net/http: request canceled (Client.Timeout exceeded while reading body) 
```
Since all AKS nodes reside in the same subnet sharing one NSG and routeTable, what could go wrong with those particular nodes?  
![NSG](/img/NSG_deny_all.png)  
Looking closely at NSG, I noticed a "DenyAll" as the default outbound rule! But that's not the actual problem coz there are plenty of "Allow" rules above it.  
But are all nodes allowed? Negative. Given that the subnet CIDR as <172.16.5.0/24>, the "Allow" rules only cover part of the whole subnet.  
And those nodes fell outside.

Takeaway: Proceed carefully when jailing your cluster nodes with NSG.

#### Bumpy DNS
What if one day your app can no longer access a middleware service like Apigee or a storage service like S3, while it's saying "I don't recognize this domain name"? It used to work happily last week!  
First please take reference from the chart in "DNS" section of [PART II](https://kuzhao.github.io/k8s/aks_networking_ii_cont/)  
Note that the DNS request from a pod is bound to land on a pod in CoreDNS service. But the next question is: where would a coreDNS pod possibly live? Well, it could be any node (preferrably in mode "system" node pool), as the coreDNS deployment does not have a node selector. Then there are two possibilities on the DNS request traffic path.
- There is a coreDNS pod on the same node as the client pod.
- CoreDNS pods live on nodes other than the client pod one.

In the second case rather than the first, cluster subnet NSG plays a part. I have seen too many examples where the customer forgot to set allowing rules for UDP traffic, while the NSG has "deny-all" at the bottom.  
A situation like above produces the outcome at the beginning of this section: certain pods cannot resolve target FQDNs intermittently; or it no longer works perfectly after a pod moves to another node.  

#### Outbound IP change
There might be a requirement by your compliance team to allow access to on-premise systems from only legit sources, including your clusters.  
Therefore, you whitelist the internet address of your pod. To mark a pass on compliance, you even put a series of resource tags on the Azure public IP for the cluster load balancer as well.  
Then after maybe three days, you suddenly notice that the on-prem access is down. More weirdly, the public IP on the load balancer changed on its own while a new one was generated.  
OK this one is a headscratcher. I will give the answer straight [here](https://docs.microsoft.com/en-us/azure/aks/faq#can-i-modify-tags-and-other-properties-of-the-aks-resources-in-the-node-resource-group). If you look closely at the public IP created along with the cluster, you should see the default tag "owner" set to "kubernetes" and "type" set to "aks-slb-managed-outbound-ip". They should not be modified.  

#### Summary
Those examples conclude this introductory series on AKS networking from my support experience. Setting them aside, we have way more kinds of "headscratcher" when Azure customers try to empower their organizations with AKS.  
You can always find me via the LinkedIn icon of this blog!