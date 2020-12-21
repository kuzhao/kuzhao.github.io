---
title: "AKS Networking Illustrated, Part II"
date: 2020-12-21T11:15:11+08:00
draft: false
---
Ok on this part, we will go over a few details on traffic in/out the cluster. Let's start where incoming traffic hits cluster services -- the load balancer.

We know from K8s concepts that a cluster service needs to be "exposed" before a client can touch it. Furthermore, the service should be exposed via either "NodePort" or "LoadBalancer" to serve external requests.  
While "NodePort" generally applies to all kinds of infrastructure, "LoadBalancer" does introduce differences at implementation. After all, there are many ways to distribute requests over backend nodes. In AKS, it is implemented through a common type of Azure resource, Azure Load Balancer.  
I love my whiteboard a lot! See the conceptual and illustrative drawing below.  
![AKS interaction with LB](/img/aks_lb_interaction.png)

A few highlights:
- Just like a common Azure LB, the LB of the cluster has all settings ranging from backend pools to health probes.
- Once an LB typed service is deployed, the kube-controller-manager talks immediately to ARM service in Azure to setup the corresponding Azure load balancer resource.
- The kube-controller-manager is responsible for analyzing incoming kubectl requests on K8s services before generating the new LB config and refreshing the LB. 

To wrap this up, AKS relies on the logic inside the cluster control plane to properly configure an Azure LB and make the traffic behavior conform to K8s service concepts.  

------  
The above concludes how K8s service loadbalancer model works out in AKS. As you can see, it is all about management operations on Azure resource config updates. But how about traffic on dataplane? To start with the answer to this question, I would like to categorize dataplane traffic into several types like below:  
![AKS dataplane traffic](/img/aks_workload_traffic.png)

While #1,2,3,5,6 follow traffic handling rules of general K8s via SNAT/DNAT, I will talk in depth about #4 which, combined with "externalTrafficPolicy" option, heavily leverages LB health probing.

**When externalTrafficPolicy=Cluster**  
This service traffic policy implies that "any nodes are capable of forwarding requests to the backend service pod". If a node does not host the pod, it will then perform a NAT before pushing out traffic to the pod address.  
Of course, LB will deem all nodes eligible to handle traffic and distribute it across them. However, note that LB will also healthcheck backend nodes while the results must be "up". And this raises my curiosity: how health probing be set by the cluster control plane?
To get into the rabbit hole and dig it through, we will first check LB healthprobing config in Azure Portal.  
![AKS LB healthcheck policy cluster](/img/lbcfg_policy_cluster.png)

OK then. What does kubectl describe service say?
```
LoadBalancer Ingress:     20.72.161.43
Port:                     http  80/TCP
TargetPort:               http/TCP
NodePort:                 http  30396/TCP
```

Apparently, the NodePort of this service is written in the healthprobing rule of LB. As long as the node with this NodePort still accepts incoming requests, the healthprobing result will be "up" and LB will forward through incoming traffic.  
With the above in mind can we get to the following topology for policy=cluster:  
![Policy=cluster](/img/policy_cluster.png)

**When externalTrafficPolicy=Local**  
In contrast to policy=cluster, this mode forbids non pod hosting nodes from handling incoming traffic. In other words, the LB should NEVER distribute traffic to those unqualifying nodes. This in turn is implemented by leveraging magical health probing.  
Similarly, Check both LB healthprobing config and kubectl describe service:  
![AKS LB healthcheck policy cluster](/img/lbcfg_policy_local.png)
```
LoadBalancer Ingress:     20.72.161.43
Port:                     http  80/TCP
TargetPort:               http/TCP
NodePort:                 http  32202/TCP
Endpoints:                172.19.10.143:8080
Session Affinity:         None
External Traffic Policy:  Local
HealthCheck NodePort:     32649
```
Here we have something interesting when compared to what we have for policy=cluster. 
- There is a new "HealthCheck Nodeport" field added to the description.
- LB healthprobing method changed from TCP:xxxx to HTTP:xxxx/healthz.
- Healthprobing port no longer takes value from NodePort but Healthcheck Nodeport.

Hence we now have also the topology for policy=local:  
![Policy=local](/img/policy_local.png)

(To be continued)