---
title: "Set timezone for a K8s Windows container"
date: 2021-07-20T11:25:11+08:00
draft: false
---
You might once be challenged by an app requirement common under Windows environment: align the timezone to its location. For a simple example, I would like my app data and logs to carry UTC+8 in East Asia, and UTC-6 in US midwest.  
Well, it's an easy job when the app runs on a physical server or VM. Just tune the timezone config of the OS. But what if you have a modern K8s cluster running Windows nodes? Also keep in mind: this cluster might well be offered by the cloud provider rather than vanilla-built by yourself.  
The OS method is then not going through in terms of both operability and scalability. In this blog, I will share with you a method effective across all cluster types -- setting it inside the app container.
### The Plan
There are three ways I can think of:
- A daemonSet with privileged securityContext, which directly sets the timezone on each windows node.
- A utility initContainer inside the app pod to set the timezone for the pod environment.
- Additional powershell command written in the app container section in the deployment manifest.

The first one sounds the best but faces a cruel fact: K8s Windows is yet to support the usage of privileged pods. Let's bet for good on this feature to come up soon.  
Per the current timezone behavior, the container only sets the info for itself. This means the second initContainer would only be effective inside that container instead of all containers in the pod. This leaves us #3 as the only one.  
### Prerequisites
So what does it take for a container to alter its timezone? As only shell commands can be run inside a Windows container, the only way is through "Set-TimeZone" powershell command.  
Assume the powershell utility functions well, there is another obstacle at the pod runtime level. This is also one of the few places where Windows nodes differ from Linux peers: the base image, usually nano-core or windows-core, has to be compatible with the OS of the node it runs on.  
You will get a good example if checking [WindowsCore DockerHub page](https://hub.docker.com/_/microsoft-windows-servercore). Scroll down to the Windows Images table under "Full Tag Listing" and you will see "OsVersion" of WindowsCore base images. In the meantime, check the OS image used by node VMs offerd by the cloud provider, i.e. Azure VMSS as below:  
![VMSS OS Image](/img/aks-vmss-img.png)  
Typically, your OS image version should be higher than the docker image version. If it's the other way around, you will get either ErrImagePulling on pod creation or a broken Set-TimeZone cmdlet due to lack of expected dll's.  
(The dll glitch did surprise me as all lib files should have been wrapped up in the docker image. Anyway, it seems Windows containers inherit system dll libs from the node OS.)
### Implementation
If your image is satisfying, then it's just a matter of properly composing the manifest. Below is my successful example, noting that "samples:aspnetapp" is built upon a compatible windowsCore base image:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample
  labels:
    app: sample
spec:
  replicas: 1
  template:
    metadata:
      name: sample
      labels:
        app: sample
    spec:
      nodeSelector:
        "beta.kubernetes.io/os": windows
      containers:
      - name: sample
        image: mcr.microsoft.com/dotnet/framework/samples:aspnetapp
        command: ['powershell','-Command','Set-TimeZone -Name "Pacific Standard Time";Get-TimeZone;ping -t localhost']
        ports:
          - containerPort: 80
  selector:
    matchLabels:
      app: sample
```
In your own implementation, replace my dummy "ping -t localhost" after timezone setting with your own image entrypoint.  
Container std output of the above deployment:
```
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.


Id                         : Pacific Standard Time
DisplayName                : (UTC-08:00) Pacific Time (US & Canada)
StandardName               : Pacific Standard Time
DaylightName               : Pacific Daylight Time
BaseUtcOffset              : -08:00:00
SupportsDaylightSavingTime : True

```
### Diagram and Summary
You can take the below goal-solution diagram as the key takeaway of this blog.  
![Goal-solution Diagram](/img/win-container-settz-diagram.png)

Always remember to confirm the node OS version and use compatible (lower) base docker images. In your production scenario, always adopt up-to-date base images as the start to build app images.  