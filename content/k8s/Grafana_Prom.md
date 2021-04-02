---
title: "How to setup a basic grafana+prometheus for a cluster"
date: 2021-04-02T09:47:11+08:00
draft: false
---
### Helm prep
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
```
### Provision prometheus through helm
```
kubectl create ns prometheus
helm install prometheus-play prometheus-community/prometheus -n prometheus
```
### Install grafana through helm
```
kubectl create ns grafana
helm install grafana-play grafana/grafana -n grafana
```
### Config in grafana
#### Grafana login
- Obtain login credential  
```
kubectl get secret grafana-ui -o yaml -n grafana
```
Extract "admin-user" and "admin-password" fields and Base64Decode. Then use them to login.

#### Dashboard
- Add prometheus data source  
 In the Grafana UI  
 Configuration (The gear icon) -> Data Sources -> Add data source -> Prometheus -> Fill "prometheus-play.prometheus" in "HTTP - URL", keep everything else by default -> Save & Test  
- Setup cluster dashboard  
 In the Grafana UI  
 Create (the icon '+' on the left sidebar) ->  Import -> Import via grafana.com -> search at https://grafana.com/grafana/dashboards and fill in the desired dashId i.e. 13770 -> Load -> Import at bottom  
- View your dash  
 In the Grafana UI  
 Dashboards -> Home -> click the dash name you just imported -> enjoy  