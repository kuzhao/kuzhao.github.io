---
title: "Running Elastic+Kibana on Azure Container Instance"
date: 2019-09-08T16:47:11+08:00
draft: false
---
### Start from [azure-quickstart-templates\201-aci-wordpress](https://github.com/Azure/azure-quickstart-templates.git)
```json
"containers": [
  {
    "name": "elasticsearch",
    "properties": {
      "image": "elasticsearch:7.3.1",
      "resources": {
        "requests": {
          "cpu": "[variables('cpuCores')]",
          "memoryInGb": "[variables('memoryInGb')]"
        }
      }
    }
  },
  {
    "name": "kibana",
    "properties": {
      "image": "kibana:7.3.1",
      "ports": [
        {
          "protocol": "Tcp",
          "port": 5601
        }
      ],
      "environmentVariables": [
        {
          "name": "ELASTICSEARCH_HOSTS",
          "value": "http://127.0.0.1:9200"
        }
      ],
      "resources": {
        "requests": {
          "cpu": "[variables('cpuCores')]",
          "memoryInGb": "[variables('memoryInGb')]"
        }
      }
    }
  }
]
```
### Failed at starting elasticsearch:
```
ERROR: [2] bootstrap checks failed  
[1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[2]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
```
**Planned solution**  
[1] <= Increase container RAM to 3GB  
[2] <= Add "environmentVariables":  
```json
"environmentVariables":[
  {
    "name": "discovery.type",
    "value": "single-node"
  }
]
```
Deployment still Failing :
```
Deployment failed. Correlation ID: 1608e8d7-3af1-4cac-bfe1-22276ccf114a. {
    "code": "InvalidContainerEnvironmentVariable",
    "message": "The environment variable name in container 'elasticsearch' of container group 'wordpress-containerinstance' is invalid. A valid environment variable name must start with alphabetic character or '_', followed by a string of alphanumeric characters or '_' (e.g. 'my_name',  or 'MY_NAME',  or 'MyName')."
}
```
### Fix envvar format\(replacing '.' with '\_'\), not working
* Elasticsearch does not honor this env as part of settings.  
* Also, cannot modify existing containerGroup:

```
Deployment failed. Correlation ID: a47f7215-8cb9-45e3-97ec-5b86aef3b465. {
  "error": {        
    "code": "InvalidContainerGroupUpdate",
    "message": "The updates on container group 'wordpress-containerinstance' are invalid. If you are going to update the os type, restart policy, network profile, CPU, memory or GPU resources for a container group, you must delete it first and then create a new one."
  }
}
```
Potential solution: pass in volume that holds elasticsearch config
