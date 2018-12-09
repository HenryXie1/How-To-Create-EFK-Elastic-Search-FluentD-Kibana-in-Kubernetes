## How To Create EFK(Elastic Search, FluentD, Kibana) in Kubernetes
### Requirement
   We need to collect ,transfer, store, analyze logs of K8S pods as well as K8S logs itself. We need to track applicatoin logs of each Pod and consider dynamic nature of containers. So we adapt EFK for that. We also plan to store log data in NFS. Refer more details in [Oracle Cloudnative Guide](https://cloudnative.oracle.com/logging.md)

### Preparation
* Install helm. Please refer official [github tutorial](https://github.com/oracle/mysql-operator/blob/master/docs/tutorial.md)
* Download efk stack config files from Github. It is based on [Oracle Cloudnative Guide](https://cloudnative.oracle.com/logging.md). However I update it to use NFS instead of local FS and change elastic search repicacount from 2 to 1 for testing. For Production, we need to use local fast filesystem and replicat be at least 2.

```
wget https://github.com/HenryXie1/How-To-Create-EFK-Elastic-Search-FluentD-Kibana-in-Kubernetes/raw/master/efk.tar.gz
tar -zxvf efk.tar.gz
```
* Create a NFS PV to Store Elastic Search Data
 * We need to create Filesystem(NFS) and mount targets in OCI first, then we can let K8S to mount them and use . Please refer official [Oracle Doc](https://docs.cloud.oracle.com/iaas/Content/File/Tasks/creatingfilesystems.htm)
 * create pv in K8S,yaml is like
 
 ```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efk-k8s-pv-name
  namespace: devops
  labels:
    app: efk-k8s
spec:
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteOnce # required
  nfs:
    server: 123.123.123.123
    path: "/EFK-Data"
 ```
 
### Install efk stack via Helm

```
$helm install efk/ --namespace devops --values efk/values.yaml
NAME:   dozing-orangutan
LAST DEPLOYED: Sat Dec  8 23:13:14 2018
NAMESPACE: devops
STATUS: DEPLOYED

RESOURCES:
==> v1beta1/Deployment
NAME                             AGE
dozing-orangutan-kibana-logging  1s

==> v1/StatefulSet
dozing-orangutan-elasticsearch-logging  1s

==> v1/Pod(related)

NAME                                              READY  STATUS             RESTARTS  AGE
dozing-orangutan-fluentd-es-5n4zp                 0/1    ContainerCreating  0         1s
dozing-orangutan-fluentd-es-5s76f                 0/1    ContainerCreating  0         1s
dozing-orangutan-fluentd-es-9zqsc                 0/1    ContainerCreating  0         1s
dozing-orangutan-kibana-logging-5bffc98f59-9f4tf  0/1    ContainerCreating  0         1s
dozing-orangutan-elasticsearch-logging-0          0/1    Pending            0         1s

==> v1/ServiceAccount

NAME                                    AGE
dozing-orangutan-fluentd-es             1s
dozing-orangutan-elasticsearch-logging  1s

==> v1/ClusterRole
dozing-orangutan-elasticsearch-logging  1s
dozing-orangutan-fluentd-es             1s

==> v1beta1/DaemonSet
dozing-orangutan-fluentd-es  1s

==> v1/ConfigMap
dozing-orangutan-fluentd-es-config  1s

==> v1/ClusterRoleBinding
dozing-orangutan-elasticsearch-logging  1s
dozing-orangutan-fluentd-es             1s

==> v1/Service
dozing-orangutan-elasticsearch-logging  1s
dozing-orangutan-kibana-logging         1s
```

Verify the pods are created well
```
$  kubectl get po -n devops
NAME                                               READY     STATUS             RESTARTS   AGE
dozing-orangutan-elasticsearch-logging-0           1/1       Running            0          4m
dozing-orangutan-fluentd-es-5n4zp                  1/1       Running            0          4m
dozing-orangutan-fluentd-es-5s76f                  1/1       Running            0          4m
dozing-orangutan-fluentd-es-9zqsc                  1/1       Running            0          4m
dozing-orangutan-kibana-logging-5bffc98f59-9f4tf   1/1       Running            0          4m
```
Verify the Service url details
```
$ kubectl get svc -n devops
NAME                                     TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
dozing-orangutan-elasticsearch-logging   ClusterIP      10.97.42.105   <none>        9200/TCP,9300/TCP   29m
dozing-orangutan-kibana-logging          LoadBalancer   10.96.6.133    <pending>     5601:32380/TCP      29m
```
As by default OCI loadbalancer is not supported in here, we update it service type to be NodePort
```
$kubectl get svc dozing-orangutan-kibana-logging -n devops -o yaml > /tmp/1.yaml
vi /tmp/1.yaml to replace LoadBalancer to be NodePort
$kubectl apply -f /tmp/1.yaml -n devops
$kubectl get svc -n devops
NAME                                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
dozing-orangutan-elasticsearch-logging   ClusterIP   10.97.42.105   <none>        9200/TCP,9300/TCP   31m
dozing-orangutan-kibana-logging          NodePort    10.96.6.133    <none>        5601:32380/TCP      31m
```
Kibana should be accessible via http://XXX.XXX.XXX.XXX:32380

### Issues
* Error start elastic search pod. Error: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
 * need to update /etc/sysctl.conf for vm.max_map_count in the node which elastic search is running. 
 ```
 #sysctl -w vm.max_map_count=262144
 ```

### Clean up
* $ helm del --purge dozing-orangutan
* $ helm ls --all dozing-orangutan  (verify it is purged)
