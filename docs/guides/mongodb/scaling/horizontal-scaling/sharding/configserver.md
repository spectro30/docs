---
title: Horizontal Scaling MongoDB Sharded Database (ConfigServer)
menu:
  docs_{{ .version }}:
    identifier: mg-horizontal-scaling-shard
    name: ConfigServer
    parent:  mg-horizontal-scaling-sharding
    weight: 20
menu_name: docs_{{ .version }}
section_menu_id: guides
---

{{< notice type="warning" message="Horizontal scaling is an Enterprise feature of KubeDB. You must have KubeDB Enterprise operator installed to test this feature." >}}

# Horizontal Scale MongoDB ConfigServer

This guide will show you how to use `KubeDB` enterprise operator to scale the configServer of a MongoDB database.

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

In this section, we are going to deploy a MongoDB sharded database. Then, in the next sections we will scale configServer of the database using `MongoDBOpsRequest` CRD. Below is the YAML of the `MongoDB` CR that we are going to create,
    
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

Let's check the number of replicas this database has from the MongoDB object, number of pods the statefulset have,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.configServer.replicas'                                                                                           11:02:09
2

$ kubectl get sts -n demo mg-sharding-configsvr -o json | jq '.spec.replicas'                                                                                                11:03:27
2
```

We can see from both command that the database has `2` replicas in the configServer. 

Also, we can verify the replicas of the configServer from an internal mongodb command by execing into a replica.

First we need to get the username and password to connect to a mongodb instance,
```console
$ kubectl get secrets -n demo mg-sharding-auth -o jsonpath='{.data.\username}' | base64 -d                                                                         11:09:51
root

$ kubectl get secrets -n demo mg-sharding-auth -o jsonpath='{.data.\password}' | base64 -d                                                                         11:10:44
xBC-EwMFivFCgUlK
```

Now let's connect to a mongodb instance and run a mongodb internal command to check the number of replicas,

```console
$ kubectl exec -n demo  mg-sharding-configsvr-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "db.adminCommand( { replSetGetStatus : 1 } ).members" --quiet     15:37:23
  
  [
  	{
  		"_id" : 0,
  		"name" : "mg-sharding-configsvr-0.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 1,
  		"stateStr" : "PRIMARY",
  		"uptime" : 388,
  		"optime" : {
  			"ts" : Timestamp(1598434641, 2),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T09:37:21Z"),
  		"syncingTo" : "",
  		"syncSourceHost" : "",
  		"syncSourceId" : -1,
  		"infoMessage" : "",
  		"electionTime" : Timestamp(1598434260, 1),
  		"electionDate" : ISODate("2020-08-26T09:31:00Z"),
  		"configVersion" : 2,
  		"self" : true,
  		"lastHeartbeatMessage" : ""
  	},
  	{
  		"_id" : 1,
  		"name" : "mg-sharding-configsvr-1.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 2,
  		"stateStr" : "SECONDARY",
  		"uptime" : 360,
  		"optime" : {
  			"ts" : Timestamp(1598434641, 2),
  			"t" : NumberLong(2)
  		},
  		"optimeDurable" : {
  			"ts" : Timestamp(1598434641, 2),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T09:37:21Z"),
  		"optimeDurableDate" : ISODate("2020-08-26T09:37:21Z"),
  		"lastHeartbeat" : ISODate("2020-08-26T09:37:24.731Z"),
  		"lastHeartbeatRecv" : ISODate("2020-08-26T09:37:23.295Z"),
  		"pingMs" : NumberLong(0),
  		"lastHeartbeatMessage" : "",
  		"syncingTo" : "mg-sharding-configsvr-0.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
  		"syncSourceHost" : "mg-sharding-configsvr-0.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
  		"syncSourceId" : 0,
  		"infoMessage" : "",
  		"configVersion" : 2
  	}
  ]
```

We can see from the above output that the configServer has 2 nodes.

We are now ready to apply the `MongoDBOpsRequest` CR to scale this database.

### Scale Up ConfigServer

Here, we are going to scale up the replicas of the ConfigServer to meet the desired number of replicas after scaling.

#### Create MongoDBOpsRequest

In order to scale up the replicas of the ConfigServer of the database, we have to create a `MongoDBOpsRequest` CR with our desired replicas. Below is the YAML of the `MongoDBOpsRequest` CR that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MongoDBOpsRequest
metadata:
  name: mops-hscale-up-configserver
  namespace: demo
spec:
  type: HorizontalScaling
  databaseRef:
    name: mg-sharding
  horizontalScaling:
    configServer:
      replicas: 3
```

Here,

- `spec.databaseRef.name` specifies that we are performing horizontal scaling operation on `mops-hscale-up-configserver` database.
- `spec.type` specifies that we are performing `HorizontalScaling` on our database.
- `spec.horizontalScaling.configServer.replicas` specifies the desired replicas after scaling.

Let's create the `MongoDBOpsRequest` CR we have shown above,

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/scaling/horizontal-scaling/mops-hscale-up-configserver.yaml
mongodbopsrequest.ops.kubedb.com/mops-hscale-up-configserver created
```

#### Verify ConfigServer replicas scaled up successfully 

If everything goes well, `KubeDB` enterprise operator will update the replicas of `MongoDB` object and related `StatefulSets` and `Pods`.

Let's wait for `MongoDBOpsRequest` to be `Successful`.  Run the following command to watch `MongoDBOpsRequest` CR,

```console
$ watch kubectl get mongodbopsrequest -n demo
Every 2.0s: kubectl get mongodbopsrequest -n demo
NAME                          TYPE                STATUS       AGE
mops-hscale-up-configserver   HorizontalScaling   Successful   2m53s
```

We can see from the above output that the `MongoDBOpsRequest` has succeeded. If we describe the `MongoDBOpsRequest` we will get an overview of the steps that were followed to scale the database.

```console
$ kubectl describe mongodbopsrequest -n demo mops-hscale-up-configserver                     
  Name:         mops-hscale-up-configserver
  Namespace:    demo
  Labels:       <none>
  Annotations:  API Version:  ops.kubedb.com/v1alpha1
  Kind:         MongoDBOpsRequest
  Metadata:
    Creation Timestamp:  2020-08-26T09:40:31Z
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
            f:configServer:
              .:
              f:replicas:
          f:type:
      Manager:      kubectl
      Operation:    Update
      Time:         2020-08-26T09:40:31Z
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
      Time:            2020-08-26T09:40:47Z
    Resource Version:  5879715
    Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/mongodbopsrequests/mops-hscale-up-configserver
    UID:               196f6dc1-89cb-41dd-8012-3aef765562c0
  Spec:
    Database Ref:
      Name:  mg-sharding
    Horizontal Scaling:
      Config Server:
        Replicas:  3
    Type:          HorizontalScaling
  Status:
    Conditions:
      Last Transition Time:  2020-08-26T09:40:31Z
      Message:               MongoDB ops request is being processed
      Observed Generation:   1
      Reason:                Scaling
      Status:                True
      Type:                  Scaling
      Last Transition Time:  2020-08-26T09:40:31Z
      Message:               Successfully paused mongodb: mg-sharding
      Observed Generation:   1
      Reason:                PauseDatabase
      Status:                True
      Type:                  PauseDatabase
      Last Transition Time:  2020-08-26T09:40:47Z
      Message:               Successfully Scaled Up Replicas of StatefulSet
      Observed Generation:   1
      Reason:                ScaleUpConfigServer 
      Status:                True
      Type:                  ScaleUpConfigServer 
      Last Transition Time:  2020-08-26T09:40:47Z
      Message:               Successfully Resumed mongodb: mg-sharding
      Observed Generation:   1
      Reason:                ResumeDatabase
      Status:                True
      Type:                  ResumeDatabase
      Last Transition Time:  2020-08-26T09:40:47Z
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
    Normal  PauseDatabase         3m15s  KubeDB Enterprise Operator  Pausing Mongodb mg-sharding in Namespace demo
    Normal  PauseDatabase         3m15s  KubeDB Enterprise Operator  Successfully Paused Mongodb mg-sharding in Namespace demo
    Normal  PauseDatabase         3m15s  KubeDB Enterprise Operator  Pausing Mongodb mg-sharding in Namespace demo
    Normal  PauseDatabase         3m15s  KubeDB Enterprise Operator  Successfully Paused Mongodb mg-sharding in Namespace demo
    Normal  ScaleUpConfigServer   2m59s  KubeDB Enterprise Operator  Successfully Scaled Up Replicas of StatefulSet
    Normal  ResumeDatabase        2m59s  KubeDB Enterprise Operator  Successfully Resumed mongodb
    Normal  Successful            2m59s  KubeDB Enterprise Operator  Successfully Scaled Database
```

Now, we are going to verify the number of replicas this database has from the MongoDB object, number of pods the statefulset have,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.configServer.replicas'                                                                                           11:02:09
3

$ kubectl get sts -n demo mg-sharding-configsvr -o json | jq '.spec.replicas'
3
```

Now let's connect to a mongodb instance and run a mongodb internal command to check the number of replicas,
```console
$ kubectl exec -n demo  mg-sharding-configsvr-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "db.adminCommand( { replSetGetStatus : 1 } ).members" --quiet
  [
  	{
  		"_id" : 0,
  		"name" : "mg-sharding-configsvr-0.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 1,
  		"stateStr" : "PRIMARY",
  		"uptime" : 1058,
  		"optime" : {
  			"ts" : Timestamp(1598435313, 2),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T09:48:33Z"),
  		"syncingTo" : "",
  		"syncSourceHost" : "",
  		"syncSourceId" : -1,
  		"infoMessage" : "",
  		"electionTime" : Timestamp(1598434260, 1),
  		"electionDate" : ISODate("2020-08-26T09:31:00Z"),
  		"configVersion" : 3,
  		"self" : true,
  		"lastHeartbeatMessage" : ""
  	},
  	{
  		"_id" : 1,
  		"name" : "mg-sharding-configsvr-1.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 2,
  		"stateStr" : "SECONDARY",
  		"uptime" : 1031,
  		"optime" : {
  			"ts" : Timestamp(1598435313, 2),
  			"t" : NumberLong(2)
  		},
  		"optimeDurable" : {
  			"ts" : Timestamp(1598435313, 2),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T09:48:33Z"),
  		"optimeDurableDate" : ISODate("2020-08-26T09:48:33Z"),
  		"lastHeartbeat" : ISODate("2020-08-26T09:48:35.250Z"),
  		"lastHeartbeatRecv" : ISODate("2020-08-26T09:48:34.860Z"),
  		"pingMs" : NumberLong(0),
  		"lastHeartbeatMessage" : "",
  		"syncingTo" : "mg-sharding-configsvr-0.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
  		"syncSourceHost" : "mg-sharding-configsvr-0.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
  		"syncSourceId" : 0,
  		"infoMessage" : "",
  		"configVersion" : 3
  	},
  	{
  		"_id" : 2,
  		"name" : "mg-sharding-configsvr-2.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
  		"health" : 1,
  		"state" : 2,
  		"stateStr" : "SECONDARY",
  		"uptime" : 460,
  		"optime" : {
  			"ts" : Timestamp(1598435313, 2),
  			"t" : NumberLong(2)
  		},
  		"optimeDurable" : {
  			"ts" : Timestamp(1598435313, 2),
  			"t" : NumberLong(2)
  		},
  		"optimeDate" : ISODate("2020-08-26T09:48:33Z"),
  		"optimeDurableDate" : ISODate("2020-08-26T09:48:33Z"),
  		"lastHeartbeat" : ISODate("2020-08-26T09:48:35.304Z"),
  		"lastHeartbeatRecv" : ISODate("2020-08-26T09:48:34.729Z"),
  		"pingMs" : NumberLong(0),
  		"lastHeartbeatMessage" : "",
  		"syncingTo" : "mg-sharding-configsvr-1.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
  		"syncSourceHost" : "mg-sharding-configsvr-1.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
  		"syncSourceId" : 1,
  		"infoMessage" : "",
  		"configVersion" : 3
  	}
  ]
```

From all the above outputs we can see that the replicas of the configServer is `3`. That means we have successfully scaled up the replicas of the MongoDB configServer replicas.


### Scale Down Replicas

Here, we are going to scale down the replicas of the ConfigServer to meet the desired number of replicas after scaling.

#### Create MongoDBOpsRequest

In order to scale down the replicas of the ConfigServer of the database, we have to create a `MongoDBOpsRequest` CR with our desired replicas. Below is the YAML of the `MongoDBOpsRequest` CR that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MongoDBOpsRequest
metadata:
  name: mops-hscale-down-configserver
  namespace: demo
spec:
  type: HorizontalScaling
  databaseRef:
    name: mg-sharding
  horizontalScaling:
    configServer:
      replicas: 2
```

Here,

- `spec.databaseRef.name` specifies that we are performing horizontal scaling operation on `mops-hscale-down-configserver` database.
- `spec.type` specifies that we are performing `HorizontalScaling` on our database.
- `spec.horizontalScaling.configServer.replicas` specifies the desired replicas after scaling.

Let's create the `MongoDBOpsRequest` CR we have shown above,

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/scaling/horizontal-scaling/mops-hscale-down-configserver.yaml
mongodbopsrequest.ops.kubedb.com/mops-hscale-down-configserver created
```

#### Verify ConfigServer replicas scaled down successfully 

If everything goes well, `KubeDB` enterprise operator will update the replicas of `MongoDB` object and related `StatefulSets` and `Pods`.

Let's wait for `MongoDBOpsRequest` to be `Successful`.  Run the following command to watch `MongoDBOpsRequest` CR,

```console
$ watch kubectl get mongodbopsrequest -n demo
Every 2.0s: kubectl get mongodbopsrequest -n demo
NAME                            TYPE                STATUS       AGE
mops-hscale-down-configserver   HorizontalScaling   Successful   82s
```

We can see from the above output that the `MongoDBOpsRequest` has succeeded. If we describe the `MongoDBOpsRequest` we will get an overview of the steps that were followed to scale the database.

```console
$ kubectl describe mongodbopsrequest -n demo mops-hscale-down-configserver                     
Name:         mops-hscale-down-configserver
Namespace:    demo
Labels:       <none>
Annotations:  API Version:  ops.kubedb.com/v1alpha1
Kind:         MongoDBOpsRequest
Metadata:
  Creation Timestamp:  2020-08-26T09:50:29Z
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
          f:configServer:
            .:
            f:replicas:
        f:type:
    Manager:      kubectl
    Operation:    Update
    Time:         2020-08-26T09:50:29Z
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
    Time:            2020-08-26T09:50:34Z
  Resource Version:  5887227
  Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/mongodbopsrequests/mops-hscale-down-configserver
  UID:               836455d9-e51a-44b2-a26a-ac8fb852b38f
Spec:
  Database Ref:
    Name:  mg-sharding
  Horizontal Scaling:
    Config Server:
      Replicas:  2
  Type:          HorizontalScaling
Status:
  Conditions:
    Last Transition Time:  2020-08-26T09:50:29Z
    Message:               MongoDB ops request is being processed
    Observed Generation:   1
    Reason:                Scaling
    Status:                True
    Type:                  Scaling
    Last Transition Time:  2020-08-26T09:50:29Z
    Message:               Successfully paused mongodb: mg-sharding
    Observed Generation:   1
    Reason:                PauseDatabase
    Status:                True
    Type:                  PauseDatabase
    Last Transition Time:  2020-08-26T09:50:34Z
    Message:               SuccessfullyScaled Down ConfigServer
    Observed Generation:   1
    Reason:                ScaleDownConfigServer 
    Status:                True
    Type:                  ScaleDownConfigServer 
    Last Transition Time:  2020-08-26T09:50:34Z
    Message:               Successfully Resumed mongodb: mg-sharding
    Observed Generation:   1
    Reason:                ResumeDatabase
    Status:                True
    Type:                  ResumeDatabase
    Last Transition Time:  2020-08-26T09:50:34Z
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
  Normal  PauseDatabase           114s  KubeDB Enterprise Operator  Pausing Mongodb mg-sharding in Namespace demo
  Normal  PauseDatabase           114s  KubeDB Enterprise Operator  Successfully Paused Mongodb mg-sharding in Namespace demo
  Normal  ScaleDownConfigServer   109s  KubeDB Enterprise Operator  Successfully Scaled Down ConfigServer
  Normal  ResumeDatabase          109s  KubeDB Enterprise Operator  Successfully Resumed mongodb
  Normal  Successful              109s  KubeDB Enterprise Operator  Successfully Scaled Database  
```

Now, we are going to verify the number of replicas this database has from the MongoDB object, number of pods the statefulset have,

```console
$ kubectl get mongodb -n demo mg-sharding -o json | jq '.spec.shardTopology.configServer.replicas'                                                                                           11:02:09
3

$ kubectl get sts -n demo mg-sharding-configsvr -o json | jq '.spec.replicas'
3
```

Now let's connect to a mongodb instance and run a mongodb internal command to check the number of replicas,
```console
$ kubectl exec -n demo  mg-sharding-configsvr-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "db.adminCommand( { replSetGetStatus : 1 } ).members" --quiet
[
	{
		"_id" : 0,
		"name" : "mg-sharding-configsvr-0.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
		"health" : 1,
		"state" : 1,
		"stateStr" : "PRIMARY",
		"uptime" : 1316,
		"optime" : {
			"ts" : Timestamp(1598435569, 1),
			"t" : NumberLong(2)
		},
		"optimeDate" : ISODate("2020-08-26T09:52:49Z"),
		"syncingTo" : "",
		"syncSourceHost" : "",
		"syncSourceId" : -1,
		"infoMessage" : "",
		"electionTime" : Timestamp(1598434260, 1),
		"electionDate" : ISODate("2020-08-26T09:31:00Z"),
		"configVersion" : 4,
		"self" : true,
		"lastHeartbeatMessage" : ""
	},
	{
		"_id" : 1,
		"name" : "mg-sharding-configsvr-1.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
		"health" : 1,
		"state" : 2,
		"stateStr" : "SECONDARY",
		"uptime" : 1288,
		"optime" : {
			"ts" : Timestamp(1598435569, 1),
			"t" : NumberLong(2)
		},
		"optimeDurable" : {
			"ts" : Timestamp(1598435569, 1),
			"t" : NumberLong(2)
		},
		"optimeDate" : ISODate("2020-08-26T09:52:49Z"),
		"optimeDurableDate" : ISODate("2020-08-26T09:52:49Z"),
		"lastHeartbeat" : ISODate("2020-08-26T09:52:52.348Z"),
		"lastHeartbeatRecv" : ISODate("2020-08-26T09:52:52.347Z"),
		"pingMs" : NumberLong(0),
		"lastHeartbeatMessage" : "",
		"syncingTo" : "mg-sharding-configsvr-0.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
		"syncSourceHost" : "mg-sharding-configsvr-0.mg-sharding-configsvr-gvr.demo.svc.cluster.local:27017",
		"syncSourceId" : 0,
		"infoMessage" : "",
		"configVersion" : 4
	}
]
```

From all the above outputs we can see that the replicas of the configServer is `2`. That means we have successfully scaled down the replicas of the MongoDB configServer replicas.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```console
kubectl delete mg -n demo mg-sharding
kubectl delete mongodbopsrequest -n demo mops-hscale-up-configserver mops-hscale-down-configserver
```