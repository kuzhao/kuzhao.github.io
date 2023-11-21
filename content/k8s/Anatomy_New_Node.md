---
title: "Anatomy of AKS Node Provisioning"
date: 2022-06-18T13:00:11+08:00
draft: false
---
Many of you have the experience of setting up a vanilla K8s cluster over your own infrastructure: Get the OS built, install kubeadm/kubelet, run kubeadm and finally setup CNI. Then to add more nodes, just run the generated "kubeadm" cmdline on each of them.  
On the contrary, will you be interested in how a node is built by a cloud provider? You will find it out in case of AKS in this article.

#### Create a node pool
Our journey starts with the birth of a new node pool via either a button click on the portal or azcli command. Normally after the process starts, you just sit and wait for a "successful" message to pop out.  
Behind the scenes, if you curiously examine the cluster's infrastructure resource group, you will see a new VMSS or availabilitySet being created. Furthermore, you will get one activity log entry as below:  
![VMSS Activity](/img/vmss_activity_create.png)  
and more that may not be quite human-readable if you click open that entry and go to "JSON" tab:  
![VMSS Create Json](/img/vmss_create_json.png)  
Replacing '\\"' with plain double quote, we have a complete resource json for the VMSS being created:
```
{
  "name": "aks-np2-39451173-vmss",
  ...
  "properties": {
    "virtualMachineProfile": {
      "osProfile": {
        ...
        "imageReference": {
          "id": "/subscriptions/109a5e88-712a-48ae-9078-9ca8b3c81345/resourceGroups/AKS-Ubuntu/providers/Microsoft.Compute/galleries/AKSUbuntu/images/1804gen2containerd/versions/2022.06.08"
        }
      },
      ...
      "extensionProfile": {
        "extensions": [
          {
            "name": "vmssCSE",
            "properties": {
              "autoUpgradeMinorVersion": true,
              "publisher": "Microsoft.Azure.Extensions",
              "type": "CustomScript",
          ...
```
OK, at least there is a hint on CustomScript extension(CSE) which is probably up to provisioning scripts.

#### Inside the Ubuntu
To dig more into further provisioning work by CSE, we have to login to a node. You can access it through either a nodeshell, or SSH with the key passed in when creating the cluster.  
**Under /opt/azure/containers**  
It's not clueless that we carefully check /opt, per the Linux convention that custom stuff can be put under that mount point.  
```bash
~# ls /opt/azure/containers
kubelet.sh  provision.complete  provision.sh  provision_cis.sh  provision_configs.sh  provision_installs.sh  provision_installs_distro.sh  provision_source.sh  provision_source_distro.sh  provision_start.sh
```

**More from cloud-init**  
Sounds surprising, but there does exist the trace of cloud-init on a node. See below:
```bash
~# ls /var/log | grep cloud-init
cloud-init-output.log
cloud-init.log
~# cat /var/log/cloud-init-output.log | grep provision | wc -l
2
~# cat /var/log/cloud-init-output.log | grep provisio
+ . /opt/azure/containers/provision_source.sh
+ . /opt/azure/containers/provision_source_distro.sh
```
Based on the above exhibits, part of node provisioning is done through cloud-init which places those scripts under /opt/azure/containers.  
But who would call and run them?

#### WAAgent and CSE
Move the cursor to syslog for more trails. As the node has just been initialized, the syslog is short and makes it easy for us to go through it.  
We do have CustomScript extension related records as below:  
```bash
Oct 10 03:13:03 aks-np2-39451173-vmss000000 python3[1401]: 2022-10-10T03:13:03.952233Z INFO ExtHandler [Microsoft.Azure.Extensions.CustomScript-2.1.7] Initializing extension Microsoft.Azure.Extensions.CustomScript-2.1.7
Oct 10 03:13:03 aks-np2-39451173-vmss000000 python3[1401]: 2022-10-10T03:13:03.953538Z INFO ExtHandler ExtHandler [CGI] Created /lib/systemd/system/azure-vmextensions-Microsoft.Azure.Extensions.CustomScript_2.1.7.slice
Oct 10 03:13:03 aks-np2-39451173-vmss000000 python3[1401]: 2022-10-10T03:13:03.954019Z INFO ExtHandler [Microsoft.Azure.Extensions.CustomScript-2.1.7] Update settings file: 1.settings
Oct 10 03:13:07 aks-np2-39451173-vmss000000 python3[1401]: time=2022-10-10T03:13:06Z version=v2.1.6/git@ad1546c-dirty operation=enable seq=1 event="creating output directory" path=/var/lib/waagent/custom-script/download/1
Oct 10 03:13:07 aks-np2-39451173-vmss000000 python3[1401]: time=2022-10-10T03:13:06Z version=v2.1.6/git@ad1546c-dirty operation=enable seq=1 event="executing command" output=/var/lib/waagent/custom-script/download/1
Oct 10 03:13:07 aks-np2-39451173-vmss000000 python3[1401]: time=2022-10-10T03:13:06Z version=v2.1.6/git@ad1546c-dirty operation=enable seq=1 event="executing protected commandToExecute" output=/var/lib/waagent/custom-script/download/1
```

So WAAgent initializes CustomScript extension first, then runs the script/command specified in 1.settings and outputs into the output directory at /var/lib/waagent/custom-script/download/1.  
The actual output file inside the directory is "stdout" (Ref [custom-script-linux_troubleshooting](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-linux#troubleshooting)). Cat the file and do a little formatting, it unveils the behavior, excerpt below:
```bash
...
+ rm -f /etc/apt/apt.conf.d/99periodic
+ [[ UBUNTU == UBUNTU ]]
+ VALIDATION_ERR=0
+ API_SERVER_CONN_RETRIES=50
+ [[ lab-aks-cn-rg-eastus-9fc4a0-8253effb.hcp.eastus.azmk8s.io == *.privatelink.* ]]
+ [[ lab-aks-cn-rg-eastus-9fc4a0-8253effb.hcp.eastus.azmk8s.io =~ ^[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+$ ]]
+ API_SERVER_DNS_RETRIES=100
+ [[ lab-aks-cn-rg-eastus-9fc4a0-8253effb.hcp.eastus.azmk8s.io == *.privatelink.* ]]
+ apt_get_purge 20 30 120 apache2-utils
...
```
Lastly, you can compare the timestamp in cloud-init-output with the CSE execution one left near the end of the stdout file. The conclusion is: cloud-init runs beforehand while CSE then wraps up the process.
