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
Replacing '\"' with plain double quote, we have a complete resource json for the VMSS being created:
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
OK, at least there is a hint on CustomScript extension which is probably up to provisioning scripts.
#### Inside the Ubuntu
To dig more into further provisioning work done automatically, we have to login on the VM. You can access it through either a nodeshell or SSH with the key passed in when creating the cluster.  
**Under /opt/azure/containers**  
It's not clueless that we carefully check /opt, per the Linux convention that custom stuff can be put under that mount point.
```bash
~# ls /opt/azure/containers
kubelet.sh  provision.complete  provision.sh  provision_cis.sh  provision_configs.sh  provision_installs.sh  provision_installs_distro.sh  provision_source.sh  provision_source_distro.sh  provision_start.sh
```
Who would be the invoker, then?  
**More on cloud-init**  
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
Based on the above exhibits, part of node provisioning is done through cloud-init which invokes tailor-made scripts under /opt/azure/containers.
#### CSE
Now is the time to chase after the custome script part we previously noticed in the Json. Although the original custom script will be deleted after finishing so we can no longer find it, it is still possible to figure out what was done throughh CSE script output.  
Per [this link](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-linux#troubleshooting), the log is /var/lib/waagent/custom-script/download/0/stdout. Cat the file and do a little formatting, it unveils the behavior, excerpt below:
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