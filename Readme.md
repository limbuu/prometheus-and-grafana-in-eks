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
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/prometheus-alertmanager-6fb6b44d8f-ghk2z         2/2     Running   0          102s
pod/prometheus-kube-state-metrics-577cdff758-tk4s2   1/1     Running   0          102s
pod/prometheus-node-exporter-8nst7                   1/1     Running   0          102s
pod/prometheus-node-exporter-nlftq                   1/1     Running   0          102s
pod/prometheus-pushgateway-5456d784bc-c6hbx          1/1     Running   0          102s
pod/prometheus-server-6c8cd688cd-m7hrf               2/2     Running   0          102s

NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/prometheus-alertmanager         ClusterIP   172.20.91.181    <none>        80/TCP     105s
service/prometheus-kube-state-metrics   ClusterIP   172.20.211.233   <none>        8080/TCP   105s
service/prometheus-node-exporter        ClusterIP   None             <none>        9100/TCP   105s
service/prometheus-pushgateway          ClusterIP   172.20.133.92    <none>        9091/TCP   105s
service/prometheus-server               ClusterIP   172.20.18.144    <none>        80/TCP     105s

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-node-exporter   2         2         2       2            2           <none>          104s

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-alertmanager         1/1     1            1           104s
deployment.apps/prometheus-kube-state-metrics   1/1     1            1           104s
deployment.apps/prometheus-pushgateway          1/1     1            1           104s
deployment.apps/prometheus-server               1/1     1            1           104s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-alertmanager-6fb6b44d8f         1         1         1       105s
replicaset.apps/prometheus-kube-state-metrics-577cdff758   1         1         1       105s
replicaset.apps/prometheus-pushgateway-5456d784bc          1         1         1       105s
replicaset.apps/prometheus-server-6c8cd688cd               1         1         1       105s
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
$ kubectl get all -n grafana
NAME                          READY   STATUS    RESTARTS   AGE
pod/grafana-b45d565bd-7795q   1/1     Running   0          2m51s

NAME              TYPE           CLUSTER-IP     EXTERNAL-IP                                                               PORT(S)        AGE
service/grafana   LoadBalancer   172.20.61.31   adfecf161cab23hshc8f3793c42c6c5d9f89c-1430368260.us-west-2.elb.amazonaws.com   80:32542/TCP   2m52s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana   1/1     1            1           2m53s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-b45d565bd   1         1         1       2m53s
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

### Grafana Dashboard 
#### 1. Cluster Monitoring Dashboard
For creating a dashboard to monitor the cluster:
* Click '+' button on left panel and select ‘Import’.
* Enter 3119 dashboard id under Grafana.com Dashboard.
* Click ‘Load’.
* Select ‘Prometheus’ as the endpoint under prometheus data sources drop down.
* Click ‘Import’.
This will show monitoring dashboard for all cluster nodes.

![alt text](https://github.com/limbuu/prometheus-and-grafana-in-eks/blob/main/images/cluster-monitoring.png)
#### 2. Pods Monitoring Dashboard
For creating a dashboard to monitor all the pods:
* Click '+' button on left panel and select ‘Import’.
* Enter 6417 dashboard id under Grafana.com Dashboard.
* Click ‘Load’.
* Enter Kubernetes Pods Monitoring as the Dashboard name.
* Click change to set the Unique identifier (uid).
* Select ‘Prometheus’ as the endpoint under prometheus data sources drop down.s
* Click ‘Import’.

![alt text](https://github.com/limbuu/prometheus-and-grafana-in-eks/blob/main/images/pods-monitoring.png)

### Uninstall Prometheus and Grafana
Run following commands to uninstall prometheus:
```
$ helm uninstall prometheus --namespace prometheus
$ kubectl delete ns prometheus

```

Run following command to uninstall grafana
```
$ helm uninstall grafana --namespace grafana
$ kubectl delete ns grafana
$ rm -rf grafana.yaml
```

## Workshop
* [Monitoring using Prometheus and Grafana](https://www.eksworkshop.com/intermediate/240_monitoring/)
