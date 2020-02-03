## Running Stateful Workloads in AKS

In this demo we will walkthrough creating an AKS cluster to host ElasticSearch.

The cluster will have 4 pools, one for management and 3 more each dedicated to a zone to host ElasticSearch.

#### Demo#1 Create an AKS cluster which spans AZs

1. Create an AKS Cluster
```shell
#define your variables
aksName=statefulWorkloads
rg=k8s
nodeSku=Standard_B2s
k8sVersion=1.14.6
logId=/subscriptions/2db66428-abf9-440c-852f-641430aa8173/resourceGroups/k8s/providers/Microsoft.OperationalInsights/workspaces/k8s-logs-workspace
k8s_version="1.15.7"
location=westeurope

#create the cluster with the management node pool 
$ az aks create \
        -g $rg \
        -n $aksName \
        --enable-addons monitoring \
        --workspace-resource-id $logId \
        --enable-cluster-autoscaler \
        --vm-set-type VirtualMachineScaleSets \
        --kubernetes-version $k8s_version \
        --load-balancer-sku standard \
        --location $location \
        --max-count 5 \
        --min-count 2 \
        --network-plugin kubenet \
        --node-count 2 \
        --node-vm-size $nodeSku \
        --nodepool-name mgmtpool 

#get the credintials 
$ az aks get-credentials --resource-group $rg --name $aksName

#verify everything is working 
$ kubectl get nodes -l agentpool=mgmtpool
NAME                               STATUS   ROLES   AGE     VERSION
aks-mgmtpool-75186649-vmss000000   Ready    agent   6m31s   v1.15.7
aks-mgmtpool-75186649-vmss000001   Ready    agent   6m30s   v1.15.7
```


2. Add 3 additional node pools, each will be dedicated for a zone with their own cluster autoscaler. 

```shell
#define the pools names as variables
Zone1PoolName=sfz1
Zone2PoolName=sfz2
Zone3PoolName=sfz3

#Provision the pools 
$ az aks nodepool add \
--resource-group $rg \
--cluster-name $aksName \
--name $Zone1PoolName \
--kubernetes-version $k8s_version \
--enable-cluster-autoscaler \
--max-count 10 \
--min-count 2 \
--node-count 2 \
--node-osdisk-size 100 \
--node-vm-size Standard_D2_v2  \
--zones 1 \
--no-wait

$ az aks nodepool add \
--resource-group $rg \
--cluster-name $aksName \
--name $Zone2PoolName \
--kubernetes-version $k8s_version \
--enable-cluster-autoscaler \
--max-count 10 \
--min-count 2 \
--node-count 2 \
--node-osdisk-size 100 \
--node-vm-size Standard_D2_v2  \
--zones 2 \
--no-wait


$ az aks nodepool add \
--resource-group $rg \
--cluster-name $aksName \
--name $Zone3PoolName \
--kubernetes-version $k8s_version \
--enable-cluster-autoscaler \
--max-count 10 \
--min-count 2 \
--node-count 2 \
--node-osdisk-size 100 \
--node-vm-size Standard_D2_v2  \
--zones 3 \
--no-wait

#wait for few minutes then verify 
$ kubectl get nodes --show-labels

```
3. Create a name space for the ElasticSearch Cluster

```shell
$ kubectl create namespace elasticsearch
```

4. Deploy ElasticSearch to your cluster 

```shell
#to ease the provisioning process we will be using Helm, we will be using the ElasticSearch Bitnami template as it comes with good defaults and high degree of flexibility

$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm fetch bitnami/elasticsearch
$ tar -xzvf elasticsearch-11.0.0.tgz && rm -rf elasticsearch-11.0.0.tgz

#explore the charts and the default values file, we will be changing this one
#copy the default values file to a new one which we will change 
$ cp elasticsearch/values.yaml values_ready.yaml 
```

5. Before we start we need to create our own storage class

```shell
# I added all the needed options in a new storage class called 'readysc' and deployed it to the cluster 
$ kubectl create -f readysc.yaml
```

6. Now we need to modify the values files so its adapts to our requirements. 

- Change the 'storageClass:' to our newly created one  'readysc'
- In the masters section change the replicas count to "3"
- In the masters section add the below affinity roles (this is required to ensure pods will land only in the node pools we provision earlier)
```yaml
  affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: failure-domain.beta.kubernetes.io/zone
                  operator: In
                  values:
                  - westeurope-1
                  - westeurope-2
                  - westeurope-3
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: agentpool
                  operator: In
                  values:
                  - sfz1
                  - sfz2
                  - sfz3
```
- in the masters section change the resource and limits to whatever you deem necessary 
- In the coordinating only we do the same changes we did in masters
- The Coordinating pods requires a service to be accessed, we will change the service type to LoadBalancer and ask for an internal one so we avoid exposing our ES to the public
```yaml
  service:
    ## coordinating-only service type
    ##
    type: LoadBalancer
    ## Elasticsearch tREST API port
    ##
    port: 9200
    ## Specify the nodePort value for the LoadBalancer and NodePort service types.
    ## ref: https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport
    ##
    # nodePort:
    ## Provide any additional annotations which may be required. This can be used to
    ## set the LoadBalancer service type to internal only.
    ## ref: https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer
    ##
    annotations: 
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```
- data pods require the same exact change as the maters 




7. So now our values are ready lets deploy elastic search to the cluster
```shell
$ helm install elasticsearch --values values_ready.yaml bitnami/elasticsearch --namespace elasticsearch 
```

8. check the pods being deployed 

```shell
# check the release in helm
$  helm ls --namespace elasticsearch
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
elasticsearch   elasticsearch   1               2020-02-03 06:13:18.871923 -0800 PST    deployed        elasticsearch-11.0.0    7.5.2      

#check that the pods ended up in the correct nodes, we can see a perfect spilt of the pods across the zones 
$ kubectl get pods -o wide -n elasticsearch
NAME                                                             READY   STATUS    RESTARTS   AGE     IP           NODE                           NOMINATED NODE   READINESS GATES
elasticsearch-elasticsearch-coordinating-only-5b6f47d94b-chdrc   1/1     Running   0          8m1s    10.244.7.3   aks-sfz1-75186649-vmss000001   <none>           <none>
elasticsearch-elasticsearch-coordinating-only-5b6f47d94b-n7qfc   1/1     Running   0          8m1s    10.244.5.3   aks-sfz3-75186649-vmss000000   <none>           <none>
elasticsearch-elasticsearch-coordinating-only-5b6f47d94b-q5khn   1/1     Running   0          8m1s    10.244.3.3   aks-sfz2-75186649-vmss000000   <none>           <none>
elasticsearch-elasticsearch-data-0                               1/1     Running   0          8m      10.244.4.3   aks-sfz3-75186649-vmss000001   <none>           <none>
elasticsearch-elasticsearch-data-1                               1/1     Running   0          6m11s   10.244.7.4   aks-sfz1-75186649-vmss000001   <none>           <none>
elasticsearch-elasticsearch-data-2                               1/1     Running   0          5m30s   10.244.3.4   aks-sfz2-75186649-vmss000000   <none>           <none>
elasticsearch-elasticsearch-master-0                             1/1     Running   0          8m      10.244.6.3   aks-sfz1-75186649-vmss000000   <none>           <none>
elasticsearch-elasticsearch-master-1                             1/1     Running   0          8m      10.244.2.4   aks-sfz2-75186649-vmss000001   <none>           <none>
elasticsearch-elasticsearch-master-2                             1/1     Running   0          8m      10.244.5.4   aks-sfz3-75186649-vmss000000   <none>           <none>


#check the clients service, which we exposed using an internal loadbalancer, the below IP will act as your ES endpoint 
kubectl get svc elasticsearch-elasticsearch-coordinating-only -n elasticsearch
NAME                                            TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
elasticsearch-elasticsearch-coordinating-only   LoadBalancer   10.0.250.46   10.240.0.12   9200:32210/TCP   10m
```

9. Spin an azure linux VM in a different subnet or in a peered vnet so we can interact with our ES cluster 
10. test if everything is fine 

```shell
#SSH to your test machine 
$ ssh -i ~/.ssh/id_rsa username@ip.of.linux.vm

#check if the cluster is OK
$ curl http://10.240.0.12:9200/_cluster/state?pretty
{
  "cluster_name" : "elastic",
  "cluster_uuid" : "3OPu6OehTWSsEq7j2LkCDg",
  "version" : 11,
  "state_uuid" : "o5791jVMQQmvpziTFL87ww",
  "master_node" : "t8neN2tbTNu7OF5T_M_OAg",
  "blocks" : { },
  "nodes" : {
    "t8neN2tbTNu7OF5T_M_OAg" : {
      "name" : "elasticsearch-elasticsearch-master-2",
      "ephemeral_id" : "P991XCwoS_6HXvWmxPwsEw",
      "transport_address" : "10.244.5.4:9300",
      "attributes" : { }
....

#put a document and search for it 
$ curl -X PUT "10.240.0.12:9200/customer/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "name": "John Doe"
}
'

$ curl -X GET "10.240.0.12:9200/customer/_doc/1?pretty"

```

11. Placeholder for scaling
12. 