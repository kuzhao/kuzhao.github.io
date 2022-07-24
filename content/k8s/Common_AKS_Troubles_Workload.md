---
title: "Common AKS Troubles - Workload"
date: 2022-07-24T15:15:11+08:00
draft: false
---
It comes to the direct concern of any K8s users, one level up from infra, on application and workload. In today's blog we are going to expand AKS problems on pod lifecycle and traffic ingress.  
#### Pod Lifecycle
The names of pod lifecycle state in AKS are of no difference with [the offcial K8s documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/). Still, a few of them are related to AKS components and even other Azure services.  
**Pending State**  
This pod state, if prolonged, means that kube-scheduler cannot find a node to fit the pod. We know per the standard K8s rule that this is caused by either resource constraints or affinity terms. There are two types of Azure disk specific reasons here.  
First, Azure VM has limitations on the number of data disks, which effectively set an upper bound on how many disk PVC a node can have for its pods.  
```
Events:
Type     Reason            Age                    From               Message
----     ------            ----                   ----               -------
Warning  FailedScheduling  16m (x275 over 4h55m)  default-scheduler  0/2 nodes are available: 1 node(s) didn't match Pod's node affinity/selector, 1 node(s) exceed max volume count.

```
Then we have "volume node affinity" term if the node pool has availability zone enabled, exhibited by the following sample event:
```
Events:   
Type     Reason            Age                From               Message  
----     ------            ----               ----               ------  
Warning  FailedScheduling  27s (x2 over 27s)  default-scheduler  0/3 nodes are available: 1 node(s) had volume node affinity conflict, 2 node(s) didn't match node selector.
```

**ContainerCreating State**  
When a pod enters this state, the container runtime on its node pulls images used in the pod before creating different containers. There were historically container runtime issues on GitHub, for example containerd, which either suspends or fails container creation. With that in mind, though, here we mainly discuss any Azure related factors instead.  
As AKS kindly offers ACR integration, a feature that allows Azure native registry attachments to a cluster, users may encounter image pulling errands with their ACRs. "ACR image pulling failure" in [Common AKS Troubles - Permission & Identity](https://kuzhao.github.io/k8s/common_aks_troubles_identity/) has well covered possible reasons.  

**CrashLoopBackoff(CLB)**  
There are countless topics on the famous CLB pod state in K8s-focused forums and GitHub. The reason is as simple as: container(s) in a pod keep getting terminated unexpectedly due to either non-zero program exit with exception or pkill done by underlying OS. [This article](https://komodor.com/learn/exit-codes-in-containers-and-kubernetes-the-complete-guide/) from komodor.com clearly marks common exit code along with their hints.  
Are there any Azure specific factors that could contribute to CLB in an AKS cluster? Seldom. There could be rare cases where certain built-in components inside kube-system keep crashing, i.e. azure-ip-masq-agent and csi-azuredisk. If that does happen, you should normally notice api-server connection failure in their pod logs. Then go down to the path of control plane connectivity troubleshooting.  
#### Ingress
Knowledge and tips for this part fall fully into the community/OSS domain. In AKS perspective, ingress controller is no more than common apps seen at the infrastructure level. The only difference is that it takes incoming web requests, setups another transport connection with upstream and then forwards them.  
The glitch related to AKS that frequently happens goes around KubeAPI version obsoletion. Since K8s 1.22, the former beta1 apiVersion "networking.k8s.io/v1beta1" fully retires, producing a breaking gap in workloads that WATCH ingress resources. If your ingress controller version is old enough to stay at probing v1beta1 only, then you will experience service failures and incomplete ingress resource status with ingress logs similar to the following:  
```bash
E0806 07:23:30.227849       6 reflector.go:138] k8s.io/client-go@v0.20.2/tools/cache/reflector.go:167: Failed to watch *v1beta1.Ingress: failed to list *v1beta1.Ingress: the server could not find the requested resource
```
The above gives clear proof that the current ingress controller no longer fits in the new cluster and KubeAPI versions. Therefore, it is then time to upgrade the ingress controller.  
