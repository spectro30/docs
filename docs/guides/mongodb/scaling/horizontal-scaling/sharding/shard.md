---
title: Horizontal Scaling MongoDB Sharded Database (Shard)
menu:
  docs_{{ .version }}:
    identifier: mg-horizontal-scaling-shard
    name: Shard
    parent:  mg-horizontal-scaling-sharding
    weight: 10
menu_name: docs_{{ .version }}
section_menu_id: guides
---

{{< notice type="warning" message="This is an Enterprise-only feature. Please install [KubeDB Enterprise Edition](/docs/setup/install/enterprise.md) to try this feature." >}}

# Horizontal Scale MongoDB Shard

This guide will show you how to use `KubeDB` enterprise operator to scale the shard of a MongoDB database.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `KubeDB` community and enterprise operator in your cluster following the steps [here]().

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

In this section, we are going to deploy a MongoDB sharded database. Then, in the next sections we will scale shards of the database using `MongoDBOpsRequest` CRD. Below is the YAML of the `MongoDB` CR that we are going to create,
    
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
mg-sharding   3.6.8-v1   Running   10m
```

Let's check the number of shards this database from the MongoDB object and the number of statefulsets it has,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.shard.shards'
3

$ kubectl get sts -n demo                                                                 
NAME                    READY   AGE
mg-sharding-configsvr   2/2     23m
mg-sharding-mongos      2/2     22m
mg-sharding-shard0      2/2     23m
mg-sharding-shard1      2/2     23m
mg-sharding-shard2      2/2     23m
```

So, We can see from the both output that the database has 3 shards.

Now, Let's check the number of replicas each shard has from the MongoDB object and the number of pod the statefulsets have,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.shard.replicas'
2

$ kubectl get sts -n demo mg-sharding-shard0 -o json | jq '.spec.replicas'  
2
```

We can see from both output that the database has 2 replicas in each shards. 

Also, we can verify the number of shard from an internal mongodb command by execing into a mongos node.

First we need to get the username and password to connect to a mongos instance,
```console
$ kubectl get secrets -n demo mg-sharding-auth -o jsonpath='{.data.\username}' | base64 -d 
root

$ kubectl get secrets -n demo mg-sharding-auth -o jsonpath='{.data.\password}' | base64 -d  
xBC-EwMFivFCgUlK
```

Now let's connect to a mongos instance and run a mongodb internal command to check the number of shards,

```console
$ kubectl exec -n demo  mg-sharding-mongos-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "sh.status()" --quiet  
--- Sharding Status --- 
 sharding version: {
    "_id" : 1,
    "minCompatibleVersion" : 5,
    "currentVersion" : 6,
    "clusterId" : ObjectId("5f45fadd48c42afd901e6265")
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

We can see from the above output that the number of shard is 3.

Also, we can verify the number of replicas each shard has from an internal mongodb command by execing into a shard node.

Now let's connect to a shard instance and run a mongodb internal command to check the number of replicas,

```console
$ kubectl exec -n demo  mg-sharding-shard0-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "db.adminCommand( { replSetGetStatus : 1 } ).members" --quiet
  [
  	{
  		"_id" : 0,
  		"name" : "mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 1,
  		"stateStr" : "PRIMARY",
  		"uptime" : 2403,
  		"optime" : {
  			"ts" : Timestamp(1598424103, 1),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T06:41:43Z"),
  		"syncingTo" : "",
  		"syncSourceHost" : "",
  		"syncSourceId" : -1,
  		"infoMessage" : "",
  		"electionTime" : Timestamp(1598421711, 1),
  		"electionDate" : ISODate("2020-08-26T06:01:51Z"),
  		"configVersion" : 2,
  		"self" : true,
  		"lastHeartbeatMessage" : ""
  	},
  	{
  		"_id" : 1,
  		"name" : "mg-sharding-shard0-1.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 2,
  		"stateStr" : "SECONDARY",
  		"uptime" : 2380,
  		"optime" : {
  			"ts" : Timestamp(1598424103, 1),
  			"t" : NumberLong(2)
  		},
  		"optimeDurable" : {
  			"ts" : Timestamp(1598424103, 1),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T06:41:43Z"),
  		"optimeDurableDate" : ISODate("2020-08-26T06:41:43Z"),
  		"lastHeartbeat" : ISODate("2020-08-26T06:41:52.541Z"),
  		"lastHeartbeatRecv" : ISODate("2020-08-26T06:41:52.259Z"),
  		"pingMs" : NumberLong(0),
  		"lastHeartbeatMessage" : "",
  		"syncingTo" : "mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"syncSourceHost" : "mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"syncSourceId" : 0,
  		"infoMessage" : "",
  		"configVersion" : 2
  	}
  ]
```

We can see from the above output that the number of replica is 2.

We are now ready to apply the `MongoDBOpsRequest` CR to update scale up and down the shards of the database.

### Scale Up

Here, we are going to scale up both the shard and their replicas to meet the desired number of replicas after scaling.

#### Create MongoDBOpsRequest

In order to scale up, we have to create a `MongoDBOpsRequest` CR with our configuration. Below is the YAML of the `MongoDBOpsRequest` CR that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MongoDBOpsRequest
metadata:
  name: mops-hscale-up-shard
  namespace: demo
spec:
  type: HorizontalScaling
  databaseRef:
    name: mg-sharding
  horizontalScaling:
    shard: 
      shards: 4
      replicas: 3
```

Here,

- `spec.databaseRef.name` specifies that we are performing horizontal scaling operation on `mops-hscale-up-shard` database.
- `spec.type` specifies that we are performing `HorizontalScaling` on our database.
- `spec.horizontalScaling.shard.shards` specifies the desired number of shards after scaling.
- `spec.horizontalScaling.shard.replicas` specifies the desired number of replicas of each shard after scaling.

Let's create the `MongoDBOpsRequest` CR we have shown above,

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/scaling/horizontal-scaling/mops-hscale-up-shard.yaml
mongodbopsrequest.ops.kubedb.com/mops-hscale-up-shard created
```

#### Verify scaling up is successful 

If everything goes well, `KubeDB` enterprise operator will update the shard and replicas of `MongoDB` object and related `StatefulSets` and `Pods`.

Let's wait for `MongoDBOpsRequest` to be `Successful`.  Run the following command to watch `MongoDBOpsRequest` CR,

```console
$ watch kubectl get mongodbopsrequest -n demo
Every 2.0s: kubectl get mongodbopsrequest -n demo
NAME                   TYPE                STATUS       AGE
mops-hscale-up-shard   HorizontalScaling   Successful   9m57s
```

We can see from the above output that the `MongoDBOpsRequest` has succeeded. If we describe the `MongoDBOpsRequest` we will get an overview of the steps that were followed to scale the database.

```console
$ kubectl describe mongodbopsrequest -n demo mops-hscale-up-shard                     
 Name:         mops-hscale-up-shard
 Namespace:    demo
 Labels:       <none>
 Annotations:  API Version:  ops.kubedb.com/v1alpha1
 Kind:         MongoDBOpsRequest
 Metadata:
   Creation Timestamp:  2020-08-26T06:44:58Z
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
           f:shard:
             .:
             f:replicas:
             f:shards:
         f:type:
     Manager:      kubectl
     Operation:    Update
     Time:         2020-08-26T06:44:58Z
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
     Time:            2020-08-26T06:49:35Z
   Resource Version:  5748416
   Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/mongodbopsrequests/mops-hscale-up-shard
   UID:               0039a5ca-cd84-44f6-9f4a-295e959244b5
 Spec:
   Database Ref:
     Name:  mg-sharding
   Horizontal Scaling:
     Shard:
       Replicas:  3
       Shards:    4
   Type:          HorizontalScaling
 Status:
   Conditions:
     Last Transition Time:  2020-08-26T06:44:58Z
     Message:               MongoDB ops request is being processed
     Observed Generation:   1
     Reason:                Scaling
     Status:                True
     Type:                  Scaling
     Last Transition Time:  2020-08-26T06:46:39Z
     Message:               Successfully Scaled Up Shards
     Observed Generation:   1
     Reason:                ScaleUpShard
     Status:                True
     Type:                  ScaleUpShard
     Last Transition Time:  2020-08-26T06:46:39Z
     Message:               Successfully paused mongodb: mg-sharding
     Observed Generation:   1
     Reason:                PauseDatabase
     Status:                True
     Type:                  PauseDatabase
     Last Transition Time:  2020-08-26T06:49:34Z
     Message:               Successfully Scaled Up Replicas of StatefulSet
     Observed Generation:   1
     Reason:                ScaleUpShardReplicas
     Status:                True
     Type:                  ScaleUpShardReplicas
     Last Transition Time:  2020-08-26T06:49:35Z
     Message:               Successfully Resumed mongodb: mg-sharding
     Observed Generation:   1
     Reason:                ResumeDatabase
     Status:                True
     Type:                  ResumeDatabase
     Last Transition Time:  2020-08-26T06:49:35Z
     Message:               Successfully completed the modification process
     Observed Generation:   1
     Reason:                Successful
     Status:                True
     Type:                  Successful
   Observed Generation:     1
   Phase:                   Successful
 Events:
   Type    Reason                Age    From                        Message
   ----    ------                ----   ----                        -------
   Normal  Progressing           10m    KubeDB Enterprise Operator  Successfully updated StatefulSets Resources
   Normal  Progressing           9m53s  KubeDB Enterprise Operator  Successfully updated StatefulSets Resources
   Normal  PauseDatabase         8m38s  KubeDB Enterprise Operator  Pausing Mongodb mg-sharding in Namespace demo
   Normal  PauseDatabase         8m38s  KubeDB Enterprise Operator  Successfully Paused Mongodb mg-sharding in Namespace demo
   Normal  ScalingUp             8m38s  KubeDB Enterprise Operator  Successfully Scaled Up Shards
   Normal  ScalingUp             7m48s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard0
   Normal  ScalingUp             7m18s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard0
   Normal  ScalingUp             7m13s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard0
   Normal  ScalingUp             7m8s   KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard1
   Normal  ScalingUp             7m8s   KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard0
   Normal  ScalingUp             6m38s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard0
   Normal  ScalingUp             6m37s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard1
   Normal  ScalingUp             6m33s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard0
   Normal  ScalingUp             6m33s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard1
   Normal  ScalingUp             6m29s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard2
   Normal  ScalingUp             5m59s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard0
   Normal  ScalingUp             5m58s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard2
   Normal  ScalingUp             5m58s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard1
   Normal  ScalingUp             5m58s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard0
   Normal  ScalingUp             5m58s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard1
   Normal  ScalingUp             5m58s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard2
   Normal  ScalingUp             5m53s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard0
   Normal  ScalingUp             5m53s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard1
   Normal  ScalingUp             5m50s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard2
   Normal  ScalingUp             5m48s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard0
   Normal  ScalingUp             5m48s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard1
   Normal  ScalingUp             5m48s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard2
   Normal  ScalingUp             5m43s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard1
   Normal  ScalingUp             5m43s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard0
   Normal  ScalingUp             5m43s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard2
   Normal  ScalingUp             5m43s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet mg-sharding-shard3
   Normal  ScaleUpShardReplicas  5m43s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet
   Normal  ResumeDatabase        5m42s  KubeDB Enterprise Operator  Resuming MongoDB
   Normal  ResumeDatabase        5m42s  KubeDB Enterprise Operator  Successfully Resumed mongodb
   Normal  Successful            5m42s  KubeDB Enterprise Operator  Successfully Scaled Database
```

Now, we are going to verify the number of shards this database has from the MongoDB object, number of statefulsets it has,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.shard.shards'         
4


$ kubectl get sts -n demo                                                                      
NAME                    READY   AGE
mg-sharding-configsvr   2/2     58m
mg-sharding-mongos      2/2     57m
mg-sharding-shard0      3/3     58m
mg-sharding-shard1      3/3     58m
mg-sharding-shard2      3/3     58m
mg-sharding-shard3      3/3     15m
```

Now let's connect to a mongos instance and run a mongodb internal command to check the number of shards,
```console
$ kubectl exec -n demo  mg-sharding-mongos-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "sh.status()" --quiet  
--- Sharding Status --- 
 sharding version: {
    "_id" : 1,
    "minCompatibleVersion" : 5,
    "currentVersion" : 6,
    "clusterId" : ObjectId("5f45fadd48c42afd901e6265")
 }
 shards:
       {  "_id" : "shard0",  "host" : "shard0/mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017,mg-sharding-shard0-1.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017,mg-sharding-shard0-2.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
       {  "_id" : "shard1",  "host" : "shard1/mg-sharding-shard1-0.mg-sharding-shard1-gvr.demo.svc.cluster.local:27017,mg-sharding-shard1-1.mg-sharding-shard1-gvr.demo.svc.cluster.local:27017,mg-sharding-shard1-2.mg-sharding-shard1-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
       {  "_id" : "shard2",  "host" : "shard2/mg-sharding-shard2-0.mg-sharding-shard2-gvr.demo.svc.cluster.local:27017,mg-sharding-shard2-1.mg-sharding-shard2-gvr.demo.svc.cluster.local:27017,mg-sharding-shard2-2.mg-sharding-shard2-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
       {  "_id" : "shard3",  "host" : "shard3/mg-sharding-shard3-0.mg-sharding-shard3-gvr.demo.svc.cluster.local:27017,mg-sharding-shard3-1.mg-sharding-shard3-gvr.demo.svc.cluster.local:27017,mg-sharding-shard3-2.mg-sharding-shard3-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
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

From all the above outputs we can see that the number of shards are `4`.

Now, we are going to verify the number of replicas each shard has from the MongoDB object, number of pods the statefulset have,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.shard.replicas'              
3

$ kubectl get sts -n demo mg-sharding-shard0 -o json | jq '.spec.replicas'          
3
```

Now let's connect to a shard instance and run a mongodb internal command to check the number of replicas,
```console
$ kubectl exec -n demo  mg-sharding-shard0-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "db.adminCommand( { replSetGetStatus : 1 } ).members" --quiet
  [
  	{
  		"_id" : 0,
  		"name" : "mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 1,
  		"stateStr" : "PRIMARY",
  		"uptime" : 3907,
  		"optime" : {
  			"ts" : Timestamp(1598425614, 1),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T07:06:54Z"),
  		"syncingTo" : "",
  		"syncSourceHost" : "",
  		"syncSourceId" : -1,
  		"infoMessage" : "",
  		"electionTime" : Timestamp(1598421711, 1),
  		"electionDate" : ISODate("2020-08-26T06:01:51Z"),
  		"configVersion" : 3,
  		"self" : true,
  		"lastHeartbeatMessage" : ""
  	},
  	{
  		"_id" : 1,
  		"name" : "mg-sharding-shard0-1.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 2,
  		"stateStr" : "SECONDARY",
  		"uptime" : 3884,
  		"optime" : {
  			"ts" : Timestamp(1598425614, 1),
  			"t" : NumberLong(2)
  		},
  		"optimeDurable" : {
  			"ts" : Timestamp(1598425614, 1),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T07:06:54Z"),
  		"optimeDurableDate" : ISODate("2020-08-26T07:06:54Z"),
  		"lastHeartbeat" : ISODate("2020-08-26T07:06:57.308Z"),
  		"lastHeartbeatRecv" : ISODate("2020-08-26T07:06:57.383Z"),
  		"pingMs" : NumberLong(0),
  		"lastHeartbeatMessage" : "",
  		"syncingTo" : "mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"syncSourceHost" : "mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"syncSourceId" : 0,
  		"infoMessage" : "",
  		"configVersion" : 3
  	},
  	{
  		"_id" : 2,
  		"name" : "mg-sharding-shard0-2.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 2,
  		"stateStr" : "SECONDARY",
  		"uptime" : 1173,
  		"optime" : {
  			"ts" : Timestamp(1598425614, 1),
  			"t" : NumberLong(2)
  		},
  		"optimeDurable" : {
  			"ts" : Timestamp(1598425614, 1),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T07:06:54Z"),
  		"optimeDurableDate" : ISODate("2020-08-26T07:06:54Z"),
  		"lastHeartbeat" : ISODate("2020-08-26T07:06:57.491Z"),
  		"lastHeartbeatRecv" : ISODate("2020-08-26T07:06:56.659Z"),
  		"pingMs" : NumberLong(0),
  		"lastHeartbeatMessage" : "",
  		"syncingTo" : "mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"syncSourceHost" : "mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"syncSourceId" : 0,
  		"infoMessage" : "",
  		"configVersion" : 3
  	}
  ]
```

From all the above outputs we can see that the replicas of each shard has is `3`. 

So, we have successfully scaled up both the shards and the shard replicas of the MongoDB database.

### Scale Down

Here, we are going to scale down both the shard and their replicas to meet the desired number of replicas after scaling.

#### Create MongoDBOpsRequest

In order to scale down, we have to create a `MongoDBOpsRequest` CR with our configuration. Below is the YAML of the `MongoDBOpsRequest` CR that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MongoDBOpsRequest
metadata:
  name: mops-hscale-down-shard
  namespace: demo
spec:
  type: HorizontalScaling
  databaseRef:
    name: mg-sharding
  horizontalScaling:
    shard: 
      shards: 3
      replicas: 2
```

Here,

- `spec.databaseRef.name` specifies that we are performing horizontal scaling operation on `mops-hscale-down-shard` database.
- `spec.type` specifies that we are performing `HorizontalScaling` on our database.
- `spec.horizontalScaling.shard.shards` specifies the desired number of shards after scaling.
- `spec.horizontalScaling.shard.replicas` specifies the desired number of replicas of each shard after scaling.

Let's create the `MongoDBOpsRequest` CR we have shown above,

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/scaling/horizontal-scaling/mops-hscale-down-shard.yaml
mongodbopsrequest.ops.kubedb.com/mops-hscale-down-shard created
```

#### Verify scaling down is successful 

If everything goes well, `KubeDB` enterprise operator will update the shards and replicas `MongoDB` object and related `StatefulSets` and `Pods`.

Let's wait for `MongoDBOpsRequest` to be `Successful`.  Run the following command to watch `MongoDBOpsRequest` CR,

```console
$ watch kubectl get mongodbopsrequest -n demo
Every 2.0s: kubectl get mongodbopsrequest -n demo
NAME                     TYPE                STATUS       AGE
mops-hscale-down-shard   HorizontalScaling   Successful   81s
```

We can see from the above output that the `MongoDBOpsRequest` has succeeded. If we describe the `MongoDBOpsRequest` we will get an overview of the steps that were followed to scale down the the database.

```console
$ kubectl describe mongodbopsrequest -n demo mops-hscale-down-shard                     
 Name:         mops-hscale-down-shard
 Namespace:    demo
 Labels:       <none>
 Annotations:  API Version:  ops.kubedb.com/v1alpha1
 Kind:         MongoDBOpsRequest
 Metadata:
   Creation Timestamp:  2020-08-26T07:10:26Z
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
           f:shard:
             .:
             f:replicas:
             f:shards:
         f:type:
     Manager:      kubectl
     Operation:    Update
     Time:         2020-08-26T07:10:26Z
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
     Time:            2020-08-26T07:10:48Z
   Resource Version:  5764724
   Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/mongodbopsrequests/mops-hscale-down-shard
   UID:               092906f7-3b6f-41c6-ad62-1ca5d9f2b2bc
 Spec:
   Database Ref:
     Name:  mg-sharding
   Horizontal Scaling:
     Shard:
       Replicas:  2
       Shards:    3
   Type:          HorizontalScaling
 Status:
   Conditions:
     Last Transition Time:  2020-08-26T07:10:26Z
     Message:               MongoDB ops request is being processed
     Observed Generation:   1
     Reason:                Scaling
     Status:                True
     Type:                  Scaling
     Last Transition Time:  2020-08-26T07:10:26Z
     Message:               Successfully paused mongodb: mg-sharding
     Observed Generation:   1
     Reason:                PauseDatabase
     Status:                True
     Type:                  PauseDatabase
     Last Transition Time:  2020-08-26T07:10:26Z
     Message:               Successfully started mongodb load balancer
     Observed Generation:   1
     Reason:                StartingBalancer
     Status:                True
     Type:                  StartingBalancer
     Last Transition Time:  2020-08-26T07:10:42Z
     Message:               Successfully Scaled Down Shard
     Observed Generation:   1
     Reason:                ScaleDownShard
     Status:                True
     Type:                  ScaleDownShard
     Last Transition Time:  2020-08-26T07:10:48Z
     Message:               Successfully Scale Down Replicas of StatefulSet
     Observed Generation:   1
     Reason:                ScaleDownShardReplicas
     Status:                True
     Type:                  ScaleDownShardReplicas
     Last Transition Time:  2020-08-26T07:10:48Z
     Message:               Successfully Resumed mongodb: mg-sharding
     Observed Generation:   1
     Reason:                ResumeDatabase
     Status:                True
     Type:                  ResumeDatabase
     Last Transition Time:  2020-08-26T07:10:48Z
     Message:               Successfully completed the modification process
     Observed Generation:   1
     Reason:                Successful
     Status:                True
     Type:                  Successful
   Observed Generation:     1
   Phase:                   Successful
 Events:
   Type    Reason                  Age   From                        Message
   ----    ------                  ----  ----                        -------
   Normal  PauseDatabase           2m4s  KubeDB Enterprise Operator  Pausing Mongodb mg-sharding in Namespace demo
   Normal  PauseDatabase           2m4s  KubeDB Enterprise Operator  Successfully Paused Mongodb mg-sharding in Namespace demo
   Normal  StartingBalancer        2m4s  KubeDB Enterprise Operator  Starting Balancer
   Normal  StartingBalancer        2m4s  KubeDB Enterprise Operator  Successfully Started Balancer
   Normal  ScaleDownShard          108s  KubeDB Enterprise Operator  Successfully Scaled Down Shard
   Normal  ScalingDown             108s  KubeDB Enterprise Operator  Scaling Down Replicas of StatefulSet
   Normal  ScaleDownShardReplicas  102s  KubeDB Enterprise Operator  Successfully Scale Down Replicas of StatefulSet
   Normal  ResumeDatabase          102s  KubeDB Enterprise Operator  Resuming MongoDB
   Normal  ResumeDatabase          102s  KubeDB Enterprise Operator  Successfully Resumed mongodb
   Normal  Successful              102s  KubeDB Enterprise Operator  Successfully Scaled Database
```

Now, we are going to verify the number of shards this database has from the MongoDB object, number of statefulsets it has,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.shard.shards'     
3

$ kubectl get sts -n demo                                                                      
NAME                    READY   AGE
mg-sharding-configsvr   2/2     78m
mg-sharding-mongos      2/2     77m
mg-sharding-shard0      2/2     78m
mg-sharding-shard1      2/2     78m
mg-sharding-shard2      2/2     78m
```

Now let's connect to a mongos instance and run a mongodb internal command to check the number of shards,
```console
$ kubectl exec -n demo  mg-sharding-mongos-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "sh.status()" --quiet  
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5f45fadd48c42afd901e6265")
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

From all the above outputs we can see that the number of shards are `3`.

Now, we are going to verify the number of replicas each shard has from the MongoDB object, number of pods the statefulset have,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.shard.replicas'                                                                            13:05:25
2

$ kubectl get sts -n demo mg-sharding-shard0 -o json | jq '.spec.replicas'                                                                                           13:05:30
2
```

Now let's connect to a shard instance and run a mongodb internal command to check the number of replicas,
```console
$ kubectl exec -n demo  mg-sharding-shard0-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "db.adminCommand( { replSetGetStatus : 1 } ).members" --quiet        13:06:31
  [
  	{
  		"_id" : 0,
  		"name" : "mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 1,
  		"stateStr" : "PRIMARY",
  		"uptime" : 4784,
  		"optime" : {
  			"ts" : Timestamp(1598426494, 1),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T07:21:34Z"),
  		"syncingTo" : "",
  		"syncSourceHost" : "",
  		"syncSourceId" : -1,
  		"infoMessage" : "",
  		"electionTime" : Timestamp(1598421711, 1),
  		"electionDate" : ISODate("2020-08-26T06:01:51Z"),
  		"configVersion" : 4,
  		"self" : true,
  		"lastHeartbeatMessage" : ""
  	},
  	{
  		"_id" : 1,
  		"name" : "mg-sharding-shard0-1.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 2,
  		"stateStr" : "SECONDARY",
  		"uptime" : 4761,
  		"optime" : {
  			"ts" : Timestamp(1598426494, 1),
  			"t" : NumberLong(2)
  		},
  		"optimeDurable" : {
  			"ts" : Timestamp(1598426494, 1),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T07:21:34Z"),
  		"optimeDurableDate" : ISODate("2020-08-26T07:21:34Z"),
  		"lastHeartbeat" : ISODate("2020-08-26T07:21:34.403Z"),
  		"lastHeartbeatRecv" : ISODate("2020-08-26T07:21:34.383Z"),
  		"pingMs" : NumberLong(0),
  		"lastHeartbeatMessage" : "",
  		"syncingTo" : "mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"syncSourceHost" : "mg-sharding-shard0-0.mg-sharding-shard0-gvr.demo.svc.cluster.local:27017",
  		"syncSourceId" : 0,
  		"infoMessage" : "",
  		"configVersion" : 4
  	}
  ]
```

From all the above outputs we can see that the replicas of each shard has is `2`. 

So, we have successfully scaled down both the shards and the shard replicas of the MongoDB database.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```console
kubectl delete mg -n demo mg-sharding
kubectl delete mongodbopsrequest -n demo mops-vscale-up-shard mops-vscale-down-shard 
```