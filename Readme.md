# Prometheus and Grafana for Monitoring EKS

## Prometheus
[Prometheus]() is open-source monitoring and alerting tool originally  build at `SoundCloud`. 

## Grafana
[Grafana](https://grafana.com/oss/grafana/) is an opens-source tool that allows you to query, visualize and alert on metrics and logs no matter where they are stored. t provides you with tools to turn your time-series database (TSDB) data into beautiful graphs and visualizations.

## Setting Up using Helm
We will use helm to add prometheus and grafana. Helm is a package manager for Kubernetes that packages multiple Kubernetes resources into single logical deployment unit call `Chart`.

### Install Helm
First, install helm CLI using script. 
```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```
You can install in other ways following this [link.](https://helm.sh/docs/intro/install/)
### Install Prometheus helm repo
Second, add prometheus helm repo
```
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
### Install Grafana helm repo
Third, add grafana helm repo
```
$ helm repo add grafana https://grafana.github.io/helm-charts
```
### Deploy Prometheus
Deploy prometheus using following command:
```
$ kubectl create namespace prometheus

$ helm install prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"
```
Now, check if Prometheus components are deployed as expected:
```
$ kubectl get all -n prometheus
```
### Deploy Grafana
To deploy grafana, first create a YAML file called `grafana.yaml`with following commands:
```
$   cat << EoF >grafana.yaml
    datasources:
    datasources.yaml:
        apiVersion: 1
        datasources:
        - name: Prometheus
        type: prometheus
        url: http://prometheus-server.prometheus.svc.cluster.local
        access: proxy
        isDefault: true
    EoF
```
Now, deploy grafana using commands:
```
$ kubectl create namespace grafana

$ helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='fuseMon123$%' \
    --values grafana.yaml \
    --set service.type=LoadBalancer
```

Check, if all grafana components are deployed as expected:
```
$ kubectl get pods all -n grafana
```
### Access Grafana
To access grafana ELB URL, run command:
```
$ export ELB=$(kubectl get svc -n grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
$ echo "http://$ELB"
```
Using the URL in browser, now you can access `Grafana Web UI.`

While logging `grafana`, use username as `admin` and for password, use the value you get after running following command:
```
$ kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

## Workshop
* [Monitoring using Prometheus and Grafana](https://www.eksworkshop.com/intermediate/240_monitoring/)
