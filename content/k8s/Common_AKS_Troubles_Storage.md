---
title: "Common AKS Troubles - Storage"
date: 2022-07-31T15:15:11+08:00
draft: false
---
We know there are two types of storage driver out of box in AKS: Azure Disk and Azure File. While we easily declare PVCs under corresponding storageClass before attach them to workloads, the underlying storage calls and mechanisms are totally different between the two.  
As the so-called "in-tree" drivers, embedded in kube-controller-manager code, have phased out, we will only discuss CSI drivers here.
#### Problems on Azure Disk
THe underlying dynamics when a PVC or a static volume is written in a deployment yaml is similar but also unique for Azure disk volumes.  
As the first step, attach-detach-controller inside kube-controller-manager captures the submission of the volume in the manifest and notifies csi-azuredisk-controller service inside the cluster control plane. In AKS, that service is put inside the managed control plane thus invisible to users.  
After getting the message, the csi controller starts making Azure API calls to create Azure disks in node resource group. It also marks the disk with the target node name, included in the message from attach-detach-controller once the pod gets scheduled. It then makes Azure calls to "attach" that disk to the target node VM. 
As the final step, csi-azuredisk-node pod (daemonSet over all AKS nodes) will notice its node has disk(s) attached during periodic polling to the controller. Once getting the result, it scans storage devices present in the OS, finds the newly attached disk LUN and mounts the filesystem on disk at the mount point for the target pod.  
**MountFailue due to timeout**  
Upon this error event in pod description, the first place for you to check is the log of container "azuredisk" in csi-azuredisk-node-xxxx pod on the same node. Normally, there should be following lines:
```bash
I0731 02:49:03.424539       1 utils.go:77] GRPC call: /csi.v1.Node/NodeStageVolume
I0731 02:49:03.424565       1 utils.go:78] GRPC request: {"publish_context":{"LUN":"0"},"staging_target_path":"/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-****/globalmount","volume_capability":{"AccessType":{"Mount":{}},"access_mode":{"mode":7}},"volume_context":{"csi.storage.k8s.io/pv/name":"pvc-****","csi.storage.k8s.io/pvc/name":"****","csi.storage.k8s.io/pvc/namespace":"lens-metrics","requestedsizegib":"19","skuname":"StandardSSD_LRS","storage.kubernetes.io/csiProvisionerIdentity":"****-disk.csi.azure.com"},"volume_id":"/subscriptions/****/resourceGroups/mc_****/providers/Microsoft.Compute/disks/pvc-****"}

```
The above means that the daemonSet csi agent is trying to mount the attached disk filesystem to he mount point for the target pod. The agent could complain that it cannot recognize either the partition table or ext4 filesystem here, the cause usually a bad Azure disk. Either data corruption or alien disk (the disk is copied from another disk) results in this.  

If nothing suspicious appears at the csi agent pod, then it worths another check over the csi controller inside the managed control plane. Although not directly accessible, you can export related logs through adding "Diagnostic Settings" for your cluster in Azure Portal:  
![AKS Diagnostic Settings](/img/aks-diag-set.jpg)  
Typical csi controller logs look like this:  
```bash
I0731 02:48:31.904096 1 azure_controller_vmss.go:121] azureDisk - update(mc_****): vm(****) - attach disk list(map[****:%!s(*provider.AttachDiskOptions=&{ReadOnly pvc-**** false 0})], %!s(*retry.Error=)) returned with %!v(MISSING)
I0731 02:49:02.325584 1 utils.go:84] GRPC response: {"publish_context":{"LUN":"0"}}
I0731 02:49:02.325563 1 azure_metrics.go:112] "Observed Request Latency" latency_seconds=31.359456259 request="azuredisk_csi_driver_controller_publish_volume" resource_group="mc_****" subscription_id="****" source="disk.csi.azure.com" volumeid="/subscriptions/****/resourceGroups/mc_****/providers/Microsoft.Compute/disks/pvc-****" node="****" result_code="succeeded"
```
You can clearly see it trying to make the call to Azure backend to attach the target disk. Keep eyes open for any Azure call failures.  
For Azure VM and disk attachment failures, the cause is normally delayed Azure compute backend responses or lack of sufficient permission on the cluster identity over Azure storage operations. You will find corresponding failed operations in "Activity" of the corresponding node VMSS before digging further on why the VMSS disk operation fails.  
#### Problems on Azure File
There are a couple of distinct differences between the processing of Azure Disk and Azure File PVC.  
**At CSI controller**  
Though the controller for Azure file share also checks and creates the Azure File resource (by provisioning a storage account), it will NOT "attach" a file share to a VM. In fact, there is no "attach" operation for Azure VM and Azure File. To use Azure file share, the guest OS must mount it to a local mount point through CIFS or NFS. (AKS Azurefile CSI driver adopts CIFS)  
Azurefile CSI controller does not perform any VMSS operations thus there will be far less problems in this part.  
**At CSI agent**  
Azure disk CSI agent does local mounting from the attached Azure disk as a block storage device. In contrast, Azurefile agent does CIFS mounting which is pure TCP traffic underneath. This key difference effectively turns any mounting issues at Azurefile agent into network troubleshooting.  
Two players behind the scenes here: TCP communication glitch (transport layer); CIFS/SMB protocol mismatch (application layer). Both can be identified by cifs-mount errCode in the error event of pod description. For example: "cifs_mount failed w/return code = -111"  
[This epic SF response](https://stackoverflow.com/a/69440916) concludes most of cifs mount errCode on a single page. The errCode number will lead you to the corresponding "#define" line with more error description. Transport layer issues concentrate in code 100-104 and 110-113, while others lead to CIFS/SMB protocol problems.  
