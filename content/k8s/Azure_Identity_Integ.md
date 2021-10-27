---
title: "An Overview of Azure Identity in AKS"
date: 2021-10-23T14:25:11+08:00
draft: false
---
Today let's talk about a confusing topic, which has not got too well documented. That is, AKS integration with Azure identity services. Try thinking of "managed AAD, pod identity, Azure RBAC, secret csi, etc" to confuse yourself.  
Before we dive into exact features, it is necessary to clarify on the **purpose** of bringing Azure identity into your cluster. With this in mind, a better distinction would be formed in your mind so as not to mix them altogether.  
- RBAC in cluster administration
This includes tasks like namespace access segregation, read/write permission on K8s objects, etc,. Basically a "who can access what" thing.  
- Auth token for workloads
This one has to do in particular with the apps running as pods. When your app makes REST api calls to external services, it needs to place in the call a bearer token recognized by the target service for authorization.  
For different purposes, we choose corresponding features from Azure identity integration.  

### Cluster administration
For this purpose we have:  
- AAD integration
- Azure RBAC integration
- Azure container registry integration
A brief table as below:  
|Feature Name | Function | How to Enable| Value|
|---------- |:---------|:----------|:---------|
|AAD integration | Retrieve AAD groups and users as K8s users | Args in AzCLI or tickbox in Portal when creating the cluster| No need to generate auth certificate per user|
|Azure RBAC | Utilize Azure user RBAC in place of K8s RBAC | Args in AzCLI for both new and existing clusters | Save effort for configure K8s RBAC |
|ACR integration | Allow the cluster to pull images from specified ACRs | Args in AzCLI for both new and existing clusters | No need to set registry credentials in K8s deploy |

### Auth token for workloads
For this purpose we have:  
- Pod identity
- Secret store csi driver
A brief table likewise:
|Feature Name | Function | How to Enable| Value|
|---------- |:---------|:----------|:---------|
|Pod identity| Make pods capable of using Azure identities | Args in AzCLI for existing clusters,or Helm(legacy) | Your apps can get bearer tokens issued by AAD |
|Secret store csi driver | Mount AKV secrets to pods as inline volumes | Args in AzCLI for both new and existing clusters,or Helm(legacy) | secrets in AKV instead of K8s secret objects, created from plaintext |

### Difference
With the above tables at hand, I believe now you can clearly distinguish one group from the other?  
++-> **AAD/RBAC/ACR integ are for cluster admins.**  
++-> **Others are for developers/users with their workloads.**  
The second key difference implicitly conveyed by their deploy/enablement methods:  
**PodIdentity/secretProvider always deploys extra K8s objects.** It's done through either Helm or AKS addon, and you will notice the difference when taking a close look at "kubectl get pod -A".  

I will find a time in the near future about in-depth comparison between features within their group.  