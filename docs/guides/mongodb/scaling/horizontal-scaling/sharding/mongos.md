---
title: Horizontal Scaling MongoDB Sharded Database (Mongos)
menu:
  docs_{{ .version }}:
    identifier: mg-horizontal-scaling-mongos
    name: Mongos
    parent:  mg-horizontal-scaling-sharding
    weight: 30
menu_name: docs_{{ .version }}
section_menu_id: guides
---

{{< notice type="warning" message="This is an Enterprise-only feature. Please install [KubeDB Enterprise Edition](/docs/setup/install/enterprise.md) to try this feature." >}}

# Horizontal Scale MongoDB Mongos

This guide will show you how to use `KubeDB` Enterprise operator to scale the mongos of a MongoDB database.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `KubeDB` Community and Enterprise operator in your cluster following the steps [here]().

- You should be familiar with the following `KubeDB` concepts:
  - [MongoDB](/docs/concepts/databases/mongodb.md)
  - [Sharding](/docs/guides/mongodb/clustering/sharding.md) 
  - [MongoDBOpsRequest](/docs/concepts/day-2-operations/mongodbopsrequest.md)
  - [Horizontal Scaling Overview](/docs/guides/mongodb/scaling/horizontal-scaling/overview.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```console
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/examples/mongodb](/docs/examples/mongodb) directory of [kubedb/docs](https://github.com/kubedb/docs) repository.

## Apply Horizontal Scaling on Sharded Database

Here, we are going to deploy a  `MongoDB` sharded database using a supported version by `KubeDB` operator. Then we are going to apply horizontal scaling on it.

### Prepare MongoDB Sharded Database

Now, we are going to deploy a `MongoDB` sharded database with version `3.6.8`.

### Deploy MongoDB Sharded Database 

In this section, we are going to deploy a MongoDB sharded database. Then, in the next sections we will scale mongos of the database using `MongoDBOpsRequest` CRD. Below is the YAML of the `MongoDB` CR that we are going to create,
    
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
$ kubectl create -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/scaling/mg-shard.yaml
mongodb.kubedb.com/mg-sharding created
```

Now, wait until `mg-sharding` has status `Running`. i.e,

```console
$ kubectl get mg -n demo                                                            
NAME          VERSION    STATUS    AGE
mg-sharding   3.6.8-v1   Running   8m
```

Let's check the number of replicas this database has from the MongoDB object, number of pods the statefulset have,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.mongos.replicas'                                                                                           11:02:09
2

$ kubectl get sts -n demo mg-sharding-mongos -o json | jq '.spec.replicas'                                                                                                11:03:27
2
```

We can see from both command that the database has `2` replicas in the mongos. 

Also, we can verify the replicas of the mongos from an internal mongodb command by execing into a replica.

First we need to get the username and password to connect to a mongodb instance,
```console
$ kubectl get secrets -n demo mg-sharding-auth -o jsonpath='{.data.\username}' | base64 -d                                                                         11:09:51
root

$ kubectl get secrets -n demo mg-sharding-auth -o jsonpath='{.data.\password}' | base64 -d                                                                         11:10:44
xBC-EwMFivFCgUlK
```

Now let's connect to a mongodb instance and run a mongodb internal command to check the number of replicas,

```console
$ kubectl exec -n demo  mg-sharding-mongos-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "sh.status()" --quiet
 --- Sharding Status --- 
    sharding version: {
    	"_id" : 1,
    	"minCompatibleVersion" : 5,
    	"currentVersion" : 6,
    	"clusterId" : ObjectId("5f463327bd21df369bb338bc")
    }
    shards:
          {  "_id" : "shard0",  "host" : "shard0/mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017,mg-sharding-shard0-1.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
          {  "_id" : "shard1",  "host" : "shard1/mg-sharding-shard1-0.mg-sharding-shard1-gvr.demo.svc.cluster.local:27017,mg-sharding-shard1-1.mg-sharding-shard1-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
          {  "_id" : "shard2",  "host" : "shard2/mg-sharding-shard2-0.mg-sharding-shard2-gvr.demo.svc.cluster.local:27017,mg-sharding-shard2-1.mg-sharding-shard2-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
    active mongoses:
          "3.6.8" : 2
    autosplit:
          Currently enabled: yes
    balancer:
          Currently enabled:  yes
          Currently running:  no
          Failed balancer rounds in last 5 attempts:  0
          Migration Results for the last 24 hours: 
                  No recent migrations
    databases:
          {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
```

We can see from the above output that the mongos has 2 active nodes.

We are now ready to apply the `MongoDBOpsRequest` CR to scale this database.

## Scale Up Mongos

Here, we are going to scale up the replicas of the Mongos to meet the desired number of replicas after scaling.

#### Create MongoDBOpsRequest

In order to scale up the replicas of the Mongos of the database, we have to create a `MongoDBOpsRequest` CR with our desired replicas. Below is the YAML of the `MongoDBOpsRequest` CR that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MongoDBOpsRequest
metadata:
  name: mops-hscale-up-mongos
  namespace: demo
spec:
  type: HorizontalScaling
  databaseRef:
    name: mg-sharding
  horizontalScaling:
    mongos:
      replicas: 3
```

Here,

- `spec.databaseRef.name` specifies that we are performing horizontal scaling operation on `mops-hscale-up-mongos` database.
- `spec.type` specifies that we are performing `HorizontalScaling` on our database.
- `spec.horizontalScaling.mongos.replicas` specifies the desired replicas after scaling.

Let's create the `MongoDBOpsRequest` CR we have shown above,

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/scaling/horizontal-scaling/mops-hscale-up-mongos.yaml
mongodbopsrequest.ops.kubedb.com/mops-hscale-up-mongos created
```

#### Verify Mongos replicas scaled up successfully 

If everything goes well, `KubeDB` Enterprise operator will update the mongos replicas of `MongoDB` object and related `StatefulSets` and `Pods`.

Let's wait for `MongoDBOpsRequest` to be `Successful`.  Run the following command to watch `MongoDBOpsRequest` CR,

```console
$ watch kubectl get mongodbopsrequest -n demo
Every 2.0s: kubectl get mongodbopsrequest -n demo
NAME                    TYPE                STATUS       AGE
mops-hscale-up-mongos   HorizontalScaling   Successful   85s
```

We can see from the above output that the `MongoDBOpsRequest` has succeeded. If we describe the `MongoDBOpsRequest` we will get an overview of the steps that were followed to scale the database.

```console
$ kubectl describe mongodbopsrequest -n demo mops-hscale-up-mongos                     
Name:         mops-hscale-up-mongos
Namespace:    demo
Labels:       <none>
Annotations:  API Version:  ops.kubedb.com/v1alpha1
Kind:         MongoDBOpsRequest
Metadata:
  Creation Timestamp:  2020-08-26T10:12:56Z
  Finalizers:
    kubedb.com
  Generation:  1
  Managed Fields:
    API Version:  ops.kubedb.com/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:databaseRef:
          .:
          f:name:
        f:horizontalScaling:
          .:
          f:mongos:
            .:
            f:replicas:
        f:type:
    Manager:      kubectl
    Operation:    Update
    Time:         2020-08-26T10:12:56Z
    API Version:  ops.kubedb.com/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:finalizers:
      f:status:
        .:
        f:conditions:
        f:observedGeneration:
        f:phase:
    Manager:         kubedb-enterprise
    Operation:       Update
    Time:            2020-08-26T10:13:11Z
  Resource Version:  5905243
  Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/mongodbopsrequests/mops-hscale-up-mongos
  UID:               f1d6efe7-634f-4764-b214-f5981ec3ed95
Spec:
  Database Ref:
    Name:  mg-sharding
  Horizontal Scaling:
    Mongos:
      Replicas:  3
  Type:          HorizontalScaling
Status:
  Conditions:
    Last Transition Time:  2020-08-26T10:12:56Z
    Message:               MongoDB ops request is being processed
    Observed Generation:   1
    Reason:                Scaling
    Status:                True
    Type:                  Scaling
    Last Transition Time:  2020-08-26T10:12:56Z
    Message:               Successfully paused mongodb: mg-sharding
    Observed Generation:   1
    Reason:                PauseDatabase
    Status:                True
    Type:                  PauseDatabase
    Last Transition Time:  2020-08-26T10:13:11Z
    Message:               Successfully Scaled Mongosmg-sharding-mongos
    Observed Generation:   1
    Reason:                ScaleMongos
    Status:                True
    Type:                  ScaleMongos
    Last Transition Time:  2020-08-26T10:13:11Z
    Message:               Successfully Resumed mongodb: mg-sharding
    Observed Generation:   1
    Reason:                ResumeDatabase
    Status:                True
    Type:                  ResumeDatabase
    Last Transition Time:  2020-08-26T10:13:11Z
    Message:               Successfully completed the modification process
    Observed Generation:   1
    Reason:                Successful
    Status:                True
    Type:                  Successful
  Observed Generation:     1
  Phase:                   Successful
Events:
  Type    Reason          Age   From                        Message
  ----    ------          ----  ----                        -------
  Normal  PauseDatabase   99s   KubeDB Enterprise Operator  Pausing Mongodb mg-sharding in Namespace demo
  Normal  PauseDatabase   99s   KubeDB Enterprise Operator  Successfully Paused Mongodb mg-sharding in Namespace demo
  Normal  PauseDatabase   99s   KubeDB Enterprise Operator  Pausing Mongodb mg-sharding in Namespace demo
  Normal  PauseDatabase   99s   KubeDB Enterprise Operator  Successfully Paused Mongodb mg-sharding in Namespace demo
  Normal  ScaleMongos     84s   KubeDB Enterprise Operator  Successfully Scaled Mongos
  Normal  ResumeDatabase  84s   KubeDB Enterprise Operator  Successfully Resumed mongodb
  Normal  Successful      84s   KubeDB Enterprise Operator  Successfully Scaled Database
```

Now, we are going to verify the number of replicas this database has from the MongoDB object, number of pods the statefulset have,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.mongos.replicas'                                                                                           11:02:09
3

$ kubectl get sts -n demo mg-sharding-mongos -o json | jq '.spec.replicas'
3
```

Now let's connect to a mongodb instance and run a mongodb internal command to check the number of replicas,
```console
$ kubectl exec -n demo  mg-sharding-mongos-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "sh.status()" --quiet
  --- Sharding Status --- 
    sharding version: {
    	"_id" : 1,
    	"minCompatibleVersion" : 5,
    	"currentVersion" : 6,
    	"clusterId" : ObjectId("5f463327bd21df369bb338bc")
    }
    shards:
          {  "_id" : "shard0",  "host" : "shard0/mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017,mg-sharding-shard0-1.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
          {  "_id" : "shard1",  "host" : "shard1/mg-sharding-shard1-0.mg-sharding-shard1-gvr.demo.svc.cluster.local:27017,mg-sharding-shard1-1.mg-sharding-shard1-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
          {  "_id" : "shard2",  "host" : "shard2/mg-sharding-shard2-0.mg-sharding-shard2-gvr.demo.svc.cluster.local:27017,mg-sharding-shard2-1.mg-sharding-shard2-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
    active mongoses:
          "3.6.8" : 3
    autosplit:
          Currently enabled: yes
    balancer:
          Currently enabled:  yes
          Currently running:  no
          Failed balancer rounds in last 5 attempts:  0
          Migration Results for the last 24 hours: 
                  No recent migrations
    databases:
          {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                  config.system.sessions
                          shard key: { "_id" : 1 }
                          unique: false
                          balancing: true
                          chunks:
                                  shard0	1
                          { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : shard0 Timestamp(1, 0)
```

From all the above outputs we can see that the replicas of the mongos is `3`. That means we have successfully scaled up the replicas of the MongoDB mongos replicas.


### Scale Down Mongos
Here, we are going to scale down the replicas of the Mongos to meet the desired number of replicas after scaling.

#### Create MongoDBOpsRequest

In order to scale down the replicas of the Mongos of the database, we have to create a `MongoDBOpsRequest` CR with our desired replicas. Below is the YAML of the `MongoDBOpsRequest` CR that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MongoDBOpsRequest
metadata:
  name: mops-hscale-down-mongos
  namespace: demo
spec:
  type: HorizontalScaling
  databaseRef:
    name: mg-sharding
  horizontalScaling:
    mongos:
      replicas: 2
```

Here,

- `spec.databaseRef.name` specifies that we are performing horizontal scaling operation on `mops-hscale-down-mongos` database.
- `spec.type` specifies that we are performing `HorizontalScaling` on our database.
- `spec.horizontalScaling.mongos.replicas` specifies the desired replicas after scaling.

Let's create the `MongoDBOpsRequest` CR we have shown above,

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/scaling/horizontal-scaling/mops-hscale-down-mongos.yaml
mongodbopsrequest.ops.kubedb.com/mops-hscale-down-mongos created
```

#### Verify Mongos replicas scaled down successfully 

If everything goes well, `KubeDB` Enterprise operator will update the mongos replicas of `MongoDB` object and related `StatefulSets` and `Pods`.

Let's wait for `MongoDBOpsRequest` to be `Successful`.  Run the following command to watch `MongoDBOpsRequest` CR,

```console
$ watch kubectl get mongodbopsrequest -n demo
Every 2.0s: kubectl get mongodbopsrequest -n demo
NAME                      TYPE                STATUS       AGE
mops-hscale-down-mongos   HorizontalScaling   Successful   53s
```

We can see from the above output that the `MongoDBOpsRequest` has succeeded. If we describe the `MongoDBOpsRequest` we will get an overview of the steps that were followed to scale the database.

```console
$ kubectl describe mongodbopsrequest -n demo mops-hscale-down-mongos                     
Name:         mops-hscale-down-mongos
Namespace:    demo
Labels:       <none>
Annotations:  API Version:  ops.kubedb.com/v1alpha1
Kind:         MongoDBOpsRequest
Metadata:
  Creation Timestamp:  2020-08-26T10:17:17Z
  Finalizers:
    kubedb.com
  Generation:  1
  Managed Fields:
    API Version:  ops.kubedb.com/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:databaseRef:
          .:
          f:name:
        f:horizontalScaling:
          .:
          f:mongos:
            .:
            f:replicas:
        f:type:
    Manager:      kubectl
    Operation:    Update
    Time:         2020-08-26T10:17:17Z
    API Version:  ops.kubedb.com/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:finalizers:
      f:status:
        .:
        f:conditions:
        f:observedGeneration:
        f:phase:
    Manager:         kubedb-enterprise
    Operation:       Update
    Time:            2020-08-26T10:17:27Z
  Resource Version:  5908548
  Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/mongodbopsrequests/mops-hscale-down-mongos
  UID:               fab6ffe1-0aab-4de2-9612-fe737c0902d6
Spec:
  Database Ref:
    Name:  mg-sharding
  Horizontal Scaling:
    Mongos:
      Replicas:  2
  Type:          HorizontalScaling
Status:
  Conditions:
    Last Transition Time:  2020-08-26T10:17:17Z
    Message:               MongoDB ops request is being processed
    Observed Generation:   1
    Reason:                Scaling
    Status:                True
    Type:                  Scaling
    Last Transition Time:  2020-08-26T10:17:17Z
    Message:               Successfully paused mongodb: mg-sharding
    Observed Generation:   1
    Reason:                PauseDatabase
    Status:                True
    Type:                  PauseDatabase
    Last Transition Time:  2020-08-26T10:17:27Z
    Message:               Successfully Scaled Mongosmg-sharding-mongos
    Observed Generation:   1
    Reason:                ScaleMongos
    Status:                True
    Type:                  ScaleMongos
    Last Transition Time:  2020-08-26T10:17:27Z
    Message:               Successfully Resumed mongodb: mg-sharding
    Observed Generation:   1
    Reason:                ResumeDatabase
    Status:                True
    Type:                  ResumeDatabase
    Last Transition Time:  2020-08-26T10:17:27Z
    Message:               Successfully completed the modification process
    Observed Generation:   1
    Reason:                Successful
    Status:                True
    Type:                  Successful
  Observed Generation:     1
  Phase:                   Successful
Events:
  Type    Reason          Age   From                        Message
  ----    ------          ----  ----                        -------
  Normal  PauseDatabase   65s   KubeDB Enterprise Operator  Pausing Mongodb mg-sharding in Namespace demo
  Normal  PauseDatabase   65s   KubeDB Enterprise Operator  Successfully Paused Mongodb mg-sharding in Namespace demo
  Normal  PauseDatabase   65s   KubeDB Enterprise Operator  Pausing Mongodb mg-sharding in Namespace demo
  Normal  PauseDatabase   65s   KubeDB Enterprise Operator  Successfully Paused Mongodb mg-sharding in Namespace demo
  Normal  ScaleMongos     55s   KubeDB Enterprise Operator  Successfully Scaled Mongos
  Normal  ResumeDatabase  55s   KubeDB Enterprise Operator  Successfully Resumed mongodb
  Normal  Successful      55s   KubeDB Enterprise Operator  Successfully Scaled Database
```

Now, we are going to verify the number of replicas this database has from the MongoDB object, number of pods the statefulset have,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.mongos.replicas'                                                                                           11:02:09
2

$ kubectl get sts -n demo mg-sharding-mongos -o json | jq '.spec.replicas'
2
```

Now let's connect to a mongodb instance and run a mongodb internal command to check the number of replicas,
```console
$ kubectl exec -n demo  mg-sharding-mongos-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "sh.status()" --quiet
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5f463327bd21df369bb338bc")
  }
  shards:
        {  "_id" : "shard0",  "host" : "shard0/mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017,mg-sharding-shard0-1.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard1",  "host" : "shard1/mg-sharding-shard1-0.mg-sharding-shard1-gvr.demo.svc.cluster.local:27017,mg-sharding-shard1-1.mg-sharding-shard1-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard2",  "host" : "shard2/mg-sharding-shard2-0.mg-sharding-shard2-gvr.demo.svc.cluster.local:27017,mg-sharding-shard2-1.mg-sharding-shard2-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
  active mongoses:
        "3.6.8" : 2
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                No recent migrations
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard0	1
                        { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : shard0 Timestamp(1, 0)
```

From all the above outputs we can see that the replicas of the mongos is `2`. That means we have successfully scaled down the replicas of the MongoDB mongos replicas.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```console
kubectl delete mg -n demo mg-sharding
kubectl delete mongodbopsrequest -n demo mops-hscale-up-mongos mops-hscale-down-mongos
```