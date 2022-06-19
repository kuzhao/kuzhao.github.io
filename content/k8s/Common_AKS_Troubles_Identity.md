---
title: "Common AKS Troubles - Permission & Identity"
date: 2022-06-19T11:15:31+08:00
draft: false
---
Though not a crucial building block of AKS, the integration with RBAC and identity services like Key Vault, Azure RBAC and managed identity could become a necessity in a strictly managed environment.  
The first step, however, is often to enable AAD integration for the cluster.  
#### AAD Integration for AKS
AAD as an authentication backend is introduced as an identity provider in the kubernetes framework (ref [here](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)). Once the AAD integration is enabled, a "guard" control plane component will be installed and handle oauth token issuance in OpenID flavor. You may catch a glimpse of it in "diagnostic settings" -> "Add new settings" on Azure portal cluster page.  
Assuming the account has been well configured in AAD with group membership, you will receive a device login code upon kubectl, along with a link to the authentication page. Instead of login success, though, you may get "login failed" due to either of below two reasons.  
**Mismatched kubectl and AKS versions**  
The output errmsg is "error: You must be logged in to the server (Unauthorized)".  
This might come out of your surprise. As long as authentication and the token have been passed and issued, how would a version difference matter?  
The answer lies in the token format within $HOME/.kube/config. If you compare the failed scenario, perhaps under kubectl v1.17, with the successful one with the latest kubectl, you will notice that the token in the latest kubectl's config removes the leading word from the one of the failed kubectl config. The thing is, there has been a change in Azure auth token related implementation at kubectl since v1.18, thus kubectl before 1.18 is no longer compatible with later versions on the matter of AAD authn token.  
**Expired service principal secret**  
This only happens to the legacy deployment of AAD integration, where you specify "aad-server-app-id" and "aad-client-app-id" on cluster creation.  
As the guard component will then rely on the principal in server-app-id, you can easily imagine what could go wrong when the principal secret gets outdated -- AAD integration stops working because the component can no longer talk to AAD with an expired secret.
*************
There are more AKS features in RBAC/identity, but most require AAD integration to be enabled as the basis.
#### Pod identity
This feature comes in handy especially for developers who could be annoyed with identity config for their apps. With the pod identity tagged to an app pod, one just needs to import Azure identity SDK library and make a oneliner call before getting a fresh bearer token for authentication purposes.  
There used to be only one way through helm to deploy AKS pod identity plugin, but it has been integrated to azcli in the middle of 2021 as an official AKS add-on. Note that both deployment methods install a "nmi" daemonSet, while the helm way additionally installs a "mic" replicaSet.  
Under the hood, mic scans and indices all identity principal added to the cluster via CRD "azureidentity" resource while nmi keeps a record of all Azure managed identity resources bound to each node (it's a daemonSet). Therefore when trouble happens, the first thing to do is to scrape and examine pod logs for both components. Error events emerge in different kinds, such as api-server dial timeout, "403" forbidden when getting AAD identites.  
**A special callout** on duplicate plugin deployments: As azcli addon method was added recently, users may still keep the old helm deployment running while enabling the new add-ons. In this case, the old helm deployment has to be cleaned up thoroughly to avoid nmi and CRD clashes.
#### ACR image pulling failure
Lastly, there is a special kind of authn/authz problem. The specialty, which is implicit, lies in the fact that the user does not directly trigger it but only notice after a normal K8s deployment.  
With the offering from AKS on the integration with private registry service, ACR, the cluster will have inherent access to an ACR registry after a simple az aks update --attach-acr command. However, one will notice an outstanding errcode "ErrImagePull" at pod status when there is a problem. When describing the affected pod, we will see either "403 forbidden" or "dial tcp timeout" in pod events. The underlying reasons, though, are totally different.  
For "**forbidden**" errors  
You should look at possible factors that block the cluster identity (either service principal or managed identity) at the registry. Double check IAM on the ACR and verify that the cluster identity has "AcrPull" role assigned. Additionally, in the case of principal as the identity, its secret must NOT expire.  
For "**timeout**"  
It usually happens on ACRs with particular network settings like ACR firewall or private endpoint. Make sure that the cluster load balancer outbound IP is whitelisted in ACR firewall settings. If a private endpoint is configured for the cluster to use, the vnet DNS of the cluster has to resolve ACR private endpoint FQDN into the right vnet IP.