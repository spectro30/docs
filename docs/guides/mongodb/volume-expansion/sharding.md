---
title: MongoDB Sharded Database Volume Expansion
menu:
  docs_{{ .version }}:
    identifier: mg-volume-expansion-shard
    name: Sharding
    parent:  mg-volume-expansion
    weight: 40
menu_name: docs_{{ .version }}
section_menu_id: guides
---

{{< notice type="warning" message="This is an Enterprise-only feature. Please install [KubeDB Enterprise Edition](/docs/setup/install/enterprise.md) to try this feature." >}}

# MongoDB Sharded Database Volume Expansion

This guide will show you how to use `KubeDB` enterprise operator to expand the volume of a MongoDB Sharded Database.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster.

- You must have a `Storage Class`, that supports volume expansion.

- Install `KubeDB` community and enterprise operator in your cluster following the steps [here]().

- You should be familiar with the following `KubeDB` concepts:
  - [MongoDB](/docs/concepts/databases/mongodb.md)
  - [Sharding](/docs/guides/mongodb/clustering/sharding.md)
  - [MongoDBOpsRequest](/docs/concepts/day-2-operations/mongodbopsrequest.md)
  - [Volume Expansion Overview](/docs/guides/mongodb/volume-expansion/overview.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```console
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/examples/mongodb](/docs/examples/mongodb) directory of [kubedb/docs](https://github.com/kubedb/docs) repository.

## Expand Volume of Sharded Database

Here, we are going to deploy a  `MongoDB` Sharded Database using a supported version by `KubeDB` operator. Then we are going to apply `MongoDBOpsRequest` to expand the volume of shard nodes and config servers.

### Prepare MongoDB Sharded Database

At first verify that your cluster has a storage class, that supports volume expansion. Let's check,

```console
$ kubectl get storageclass                                                                                                                                           20:22:33
NAME                 PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   kubernetes.io/gce-pd   Delete          Immediate           true                   2m49s
```

We can see from the output the `standard` storage class has `ALLOWVOLUMEEXPANSION` field as true. So, this storage class supports volume expansion. We can use it.

Now, we are going to deploy a `MongoDB` standalone database with version `3.6.8`.

### Deploy MongoDB

In this section, we are going to deploy a MongoDB Sharded database with 1GB volume for each of the shard nodes and config servers. Then, in the next sections we will expand the volume of shard nodes and config servers to 2GB using `MongoDBOpsRequest` CRD. Below is the YAML of the `MongoDB` CR that we are going to create,

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mg-sharding
  namespace: demo
spec:
  version: 3.6.8-v1
  shardTopology:
    configServer:
      replicas: 2
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: standard
    mongos:
      replicas: 2
    shard:
      replicas: 2
      shards: 3
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: standard
```

Let's create the `MongoDB` CR we have shown above,

```console
$ kubectl create -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/volume-expansion/mg-shard.yaml
mongodb.kubedb.com/mg-sharding created
```

Now, wait until `mg-sharding` has status `Running`. i.e,

```console
$ kubectl get mg -n demo                                                               
NAME          VERSION    STATUS    AGE
mg-sharding   3.6.8-v1   Running   2m45s
```

Let's check volume size from statefulset, and from the persistent volume of shards and config servers,

```console
$ kubectl get sts -n demo mg-sharding-configsvr -o json | jq '.spec.volumeClaimTemplates[].spec.resources.requests.storage'
"1Gi"

$ kubectl get sts -n demo mg-sharding-shard0 -o json | jq '.spec.volumeClaimTemplates[].spec.resources.requests.storage'
"1Gi"

$ kubectl get pv -n demo                                                                                          
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                  STORAGECLASS   REASON   AGE
pvc-06989d55-7b35-471e-872e-ed1d57264799   1Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard1-1      standard                4m2s
pvc-22bcb386-f3ca-4b5f-8de5-25547246169a   1Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard2-1      standard                3m58s
pvc-56a41bc4-24fa-4edb-a00f-5e131d3445e0   1Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard0-1      standard                4m9s
pvc-666cd25a-35b0-4973-86b8-c9ec4eb94b4f   1Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard0-0      standard                4m53s
pvc-a68ba7f0-b041-49f4-9e80-edac58dcd148   1Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-configsvr-1   standard                4m3s
pvc-c75fa730-a62c-4b12-8d52-4547b394cbd8   1Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard2-0      standard                4m54s
pvc-d18c1112-f81f-4ed2-9afd-6d8344cd0a29   1Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard1-0      standard                4m53s
pvc-deca2307-6c02-4ca6-84b9-461995ec394d   1Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-configsvr-0   standard                4m53s
```

You can see the statefulsets have 1GB storage, and the capacity of all the persistent volumes are also 1GB.

We are now ready to apply the `MongoDBOpsRequest` CR to expand the volume of this database.

### Volume Expansion of Shard Nodes

Here, we are going to expand the volume of the shard nodes of the database.

#### Create MongoDBOpsRequest

In order to expand the volume of the shard nodes of the database, we have to create a `MongoDBOpsRequest` CR with our desired volume size. Below is the YAML of the `MongoDBOpsRequest` CR that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MongoDBOpsRequest
metadata:
  name: mops-volume-exp-shard
  namespace: demo
spec:
  type: VolumeExpansion
  databaseRef:
    name: mg-sharding
  volumeExpansion:
    shard: 2Gi
```

Here,

- `spec.databaseRef.name` specifies that we are performing volume expansion operation on `mops-volume-exp-shard` database.
- `spec.type` specifies that we are performing `VolumeExpansion` on our database.
- `spec.volumeExpansion.shard` specifies the desired volume size.

Let's create the `MongoDBOpsRequest` CR we have shown above,

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/volume-expansion/mops-volume-exp-shard.yaml
mongodbopsrequest.ops.kubedb.com/mops-volume-exp-shard created
```

#### Verify MongoDB shard volumes expanded successfully 

If everything goes well, `KubeDB` enterprise operator will update the volume size of `MongoDB` object and related `StatefulSets` and `Persistent Volumes`.

Let's wait for `MongoDBOpsRequest` to be `Successful`.  Run the following command to watch `MongoDBOpsRequest` CR,

```console
$ kubectl get mongodbopsrequest -n demo
Every 2.0s: kubectl get mongodbopsrequest -n demo
NAME                    TYPE              STATUS       AGE
mops-volume-exp-shard   VolumeExpansion   Successful   3m49s
```

We can see from the above output that the `MongoDBOpsRequest` has succeeded. If we describe the `MongoDBOpsRequest` we will get an overview of the steps that were followed to expand the volume of the database.

```console
$ kubectl describe mongodbopsrequest -n demo mops-volume-exp-shard   
Name:         mops-volume-exp-shard
Namespace:    demo
Labels:       <none>
Annotations:  API Version:  ops.kubedb.com/v1alpha1
Kind:         MongoDBOpsRequest
Metadata:
  Creation Timestamp:  2020-08-26T04:10:45Z
  Finalizers:
    kubedb.com
  Generation:        1
  Resource Version:  284155
  Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/mongodbopsrequests/mops-volume-exp-shard
  UID:               9278a2d7-d806-4f5d-9d68-99f7845d19ad
Spec:
  Database Ref:
    Name:  mg-sharding
  Type:    VolumeExpansion
  Volume Expansion:
    Shard:  2Gi
Status:
  Conditions:
    Last Transition Time:  2020-08-26T04:10:45Z
    Message:               MongoDB ops request is being processed
    Observed Generation:   1
    Reason:                Scaling
    Status:                True
    Type:                  Scaling
    Last Transition Time:  2020-08-26T04:13:45Z
    Message:               Successfully updated Storage
    Observed Generation:   1
    Reason:                VolumeExpansion
    Status:                True
    Type:                  VolumeExpansion
    Last Transition Time:  2020-08-26T04:13:45Z
    Message:               Successfully Resumed mongodb: mg-sharding
    Observed Generation:   1
    Reason:                ResumeDatabase
    Status:                True
    Type:                  ResumeDatabase
    Last Transition Time:  2020-08-26T04:13:45Z
    Message:               Successfully completed the modification process
    Observed Generation:   1
    Reason:                Successful
    Status:                True
    Type:                  Successful
  Observed Generation:     1
  Phase:                   Successful
Events:
  Type    Reason           Age   From                        Message
  ----    ------           ----  ----                        -------
  Normal  VolumeExpansion  79s   KubeDB Enterprise Operator  Successfully Updated Storage
  Normal  ResumeDatabase   79s   KubeDB Enterprise Operator  Resuming MongoDB
  Normal  ResumeDatabase   79s   KubeDB Enterprise Operator  Successfully Resumed mongodb
  Normal  Successful       79s   KubeDB Enterprise Operator  Successfully Scaled Database
```

Now, we are going to verify from the `Statefulset`, and the `Persistent Volumes` whether the volume of the database has expanded to meet the desired state, Let's check,

```console
$ kubectl get sts -n demo mg-sharding-shard0 -o json | jq '.spec.volumeClaimTemplates[].spec.resources.requests.storage'                                             10:15:51
"2Gi"

$ kubectl get pv -n demo                                                                                          
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                  STORAGECLASS   REASON   AGE
pvc-06989d55-7b35-471e-872e-ed1d57264799   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard1-1      standard                12m
pvc-22bcb386-f3ca-4b5f-8de5-25547246169a   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard2-1      standard                12m
pvc-56a41bc4-24fa-4edb-a00f-5e131d3445e0   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard0-1      standard                12m
pvc-666cd25a-35b0-4973-86b8-c9ec4eb94b4f   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard0-0      standard                13m
pvc-a68ba7f0-b041-49f4-9e80-edac58dcd148   1Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-configsvr-1   standard                12m
pvc-c75fa730-a62c-4b12-8d52-4547b394cbd8   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard2-0      standard                13m
pvc-d18c1112-f81f-4ed2-9afd-6d8344cd0a29   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard1-0      standard                13m
pvc-deca2307-6c02-4ca6-84b9-461995ec394d   1Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-configsvr-0   standard                13m
```

The above output verifies that we have successfully expanded the volume of the shard nodes of the MongoDB database.


### Volume Expansion of ConfigServer Nodes

Here, we are going to expand the volume of the config servers of the database.

#### Create MongoDBOpsRequest

In order to expand the volume of the config servers of the database, we have to create a `MongoDBOpsRequest` CR with our desired volume size. Below is the YAML of the `MongoDBOpsRequest` CR that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MongoDBOpsRequest
metadata:
  name: mops-volume-exp-configserver
  namespace: demo
spec:
  type: VolumeExpansion
  databaseRef:
    name: mg-sharding
  volumeExpansion:
    configServer: 2Gi
```

Here,

- `spec.databaseRef.name` specifies that we are performing volume expansion operation on `mops-volume-exp-configserver` database.
- `spec.type` specifies that we are performing `VolumeExpansion` on our database.
- `spec.volumeExpansion.configServer` specifies the desired volume size.

Let's create the `MongoDBOpsRequest` CR we have shown above,

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/volume-expansion/mops-volume-exp-configsvr.yaml
mongodbopsrequest.ops.kubedb.com/mops-volume-exp-configserver created
```

#### Verify MongoDB config server volumes expanded successfully 

If everything goes well, `KubeDB` enterprise operator will update the volume size of `MongoDB` object and related `StatefulSets` and `Persistent Volumes`.

Let's wait for `MongoDBOpsRequest` to be `Successful`.  Run the following command to watch `MongoDBOpsRequest` CR,

```console
$ kubectl get mongodbopsrequest -n demo
Every 2.0s: kubectl get mongodbopsrequest -n demo
NAME                           TYPE              STATUS       AGE
mops-volume-exp-configserver   VolumeExpansion   Successful   111s
```

We can see from the above output that the `MongoDBOpsRequest` has succeeded. If we describe the `MongoDBOpsRequest` we will get an overview of the steps that were followed to expand the volume of the database.

```console
$ kubectl describe mongodbopsrequest -n demo mops-volume-exp-configserver   
Name:         mops-volume-exp-configserver
Namespace:    demo
Labels:       <none>
Annotations:  API Version:  ops.kubedb.com/v1alpha1
Kind:         MongoDBOpsRequest
Metadata:
  Creation Timestamp:  2020-08-26T04:18:58Z
  Finalizers:
    kubedb.com
  Generation:        1
  Resource Version:  286472
  Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/mongodbopsrequests/mops-volume-exp-configserver
  UID:               de381f2c-1e80-4a76-9ccb-947245f66540
Spec:
  Database Ref:
    Name:  mg-sharding
  Type:    VolumeExpansion
  Volume Expansion:
    Config Server:  2Gi
Status:
  Conditions:
    Last Transition Time:  2020-08-26T04:18:58Z
    Message:               MongoDB ops request is being processed
    Observed Generation:   1
    Reason:                Scaling
    Status:                True
    Type:                  Scaling
    Last Transition Time:  2020-08-26T04:20:28Z
    Message:               Successfully updated Storage
    Observed Generation:   1
    Reason:                VolumeExpansion
    Status:                True
    Type:                  VolumeExpansion
    Last Transition Time:  2020-08-26T04:20:28Z
    Message:               Successfully Resumed mongodb: mg-sharding
    Observed Generation:   1
    Reason:                ResumeDatabase
    Status:                True
    Type:                  ResumeDatabase
    Last Transition Time:  2020-08-26T04:20:28Z
    Message:               Successfully completed the modification process
    Observed Generation:   1
    Reason:                Successful
    Status:                True
    Type:                  Successful
  Observed Generation:     1
  Phase:                   Successful
Events:
  Type    Reason           Age   From                        Message
  ----    ------           ----  ----                        -------
  Normal  VolumeExpansion  49s   KubeDB Enterprise Operator  Successfully Updated Storage
  Normal  ResumeDatabase   49s   KubeDB Enterprise Operator  Resuming MongoDB
  Normal  ResumeDatabase   49s   KubeDB Enterprise Operator  Successfully Resumed mongodb
  Normal  Successful       49s   KubeDB Enterprise Operator  Successfully Scaled Database
```

Now, we are going to verify from the `Statefulset`, and the `Persistent Volumes` whether the volume of the database has expanded to meet the desired state, Let's check,

```console
$ kubectl get sts -n demo mg-sharding-configsvr -o json | jq '.spec.volumeClaimTemplates[].spec.resources.requests.storage'                                             10:15:51
"2Gi"

$ kubectl get pv -n demo                                                                                          
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                  STORAGECLASS   REASON   AGE
pvc-06989d55-7b35-471e-872e-ed1d57264799   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard1-1      standard                17m
pvc-22bcb386-f3ca-4b5f-8de5-25547246169a   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard2-1      standard                17m
pvc-56a41bc4-24fa-4edb-a00f-5e131d3445e0   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard0-1      standard                17m
pvc-666cd25a-35b0-4973-86b8-c9ec4eb94b4f   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard0-0      standard                18m
pvc-a68ba7f0-b041-49f4-9e80-edac58dcd148   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-configsvr-1   standard                17m
pvc-c75fa730-a62c-4b12-8d52-4547b394cbd8   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard2-0      standard                18m
pvc-d18c1112-f81f-4ed2-9afd-6d8344cd0a29   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-shard1-0      standard                18m
pvc-deca2307-6c02-4ca6-84b9-461995ec394d   2Gi        RWO            Delete           Bound    demo/datadir-mg-sharding-configsvr-0   standard                18m
```

The above output verifies that we have successfully expanded the volume of the config servers of the MongoDB database.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```console
kubectl delete mg -n demo mg-sharding
kubectl delete mongodbopsrequest -n demo mops-volume-exp-shard mops-volume-exp-configserver
```