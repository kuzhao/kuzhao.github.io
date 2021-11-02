---
title: "How to setup a vanilla Calico cluster(RH based)"
date: 2021-08-12T14:47:11+08:00
draft: false
---
## Setup the OS 
**Add K8s repo**  
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubectl
```
**(if within China GFW)**  
Reference: https://xuxinkun.github.io/2019/06/11/cn-registry/
```bash
# CentOS/RHEL/Fedora
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
**Setup docker daemon**
```bash
# Disable SELinux -- set to permissive mode
setenforce 0
yum install -y docker
# Disable selinux in docker daemon cfg
sed -i 's/--selinux-enabled//' /etc/sysconfig/docker
systemctl enable docker --now
```
## Provision the cluster
**Setup K8s master**  
Prepare kubeadm init.yml. Ref: https://github.com/maguowei/gotok8s/blob/master/init.yml  
After that, **Set kubernetesVersion within the yml.**  

Install kubelet and kubeadm of the same kubernetesVersion with the yml.
```bash
yum install -y kubeadm-1.xx.x kubelet-1.xx.x
systemctl enable kubelet --now
```
Use kubeadm to init K8s master.
```bash
kubeadm init --config init.yml
```
You should then have both the kubeconfig file and the "kubeadm join" command from the output. **Do NOT lose them.** You will need it for later kubectl exec.  

**Setup agent nodes**  
On each agent node:  
```bash
yum install -y kubeadm-1.xx.x kubelet-1.xx.x
systemctl enable kubelet --now
kubeadm join xxxxxxxxx
```
**Isolate the master node**  
This is to purge existing workloads, and further prevent pod scheduling on master.
```bash
kubectl cordon <master-node-name>
kubectl get pod -A -o wide
# Optional below
kubectl delete pod <corednsPod if On Master> 
```
## Setup Calico
Follow https://docs.projectcalico.org/getting-started/kubernetes/quickstart#install-calico but **Halt before step#2**.  
At #2, replace "custom-resources.yaml" from calico.org with one similar with below:  
```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 24
      cidr: 10.244.1.0/24
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
```
Then proceed with kubectl create -f in #2.
## Deploy the traffic testsuit
Topology:  
CurlClient <-> WebServer  
node #1        node #2

On node #2, prepare a test file for curl download from node #1. Place it under /mnt/resource/test.  
Use the below manifest and fill node names to deploy both the client and the server:  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        args:
        - -token
        - f9403fc5f537b4ab332d
        - -upload_limit
        - "100000000"
        - /tmp
        image: mayth/simple-upload-server
        volumeMounts:
        - name: test-dnld
          mountPath: /tmp/test.bin
        ports:
        - name: http
          containerPort: 25478
      volumes:
      - name: test-dnld
        hostPath:
          path: /mnt/resource/test/test.bin
      nodeName: <Node #2 Name>
---
kind: Service
apiVersion: v1
metadata:
  name: service-test
spec:
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: http
---
apiVersion: v1
kind: Pod
metadata:
  name: test-dnld
spec:
  containers:
  - name: ubuntu
    image: ubuntu:latest
    command:
    - /bin/bash
    args:
    - -c
    - apt-get update;apt-get install -y curl;while true;do curl http://service-test/files/test.bin?token=f9403fc5f537b4ab332d -o echo.bin;curl -Ffile=@echo.bin http://service-test/upload?token=f9403fc5f537b4ab332d;sleep 5;done
  nodeName: <Node #1 Name>
```
