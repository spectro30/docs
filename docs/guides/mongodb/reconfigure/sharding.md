---
title: MongoDB Reconfigure Sharded Database
menu:
  docs_{{ .version }}:
    identifier: mg-reconfigure-shard
    name: Reconfigure Shard
    parent:  mg-reconfigure
    weight: 40
menu_name: docs_{{ .version }}
section_menu_id: guides
---

{{< notice type="warning" message="Reconfiguring database is an Enterprise feature of KubeDB. You must have KubeDB Enterprise operator installed to test this feature." >}}

# Reconfigure MongoDB Shard

This guide will show you how to use `KubeDB` enterprise operator to reconfigure a MongoDB shard.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster.

- Install `KubeDB` community and enterprise operator in your cluster following the steps [here]().

- You should be familiar with the following `KubeDB` concepts:
  - [MongoDB](/docs/concepts/databases/mongodb.md)
  - [Sharding](/docs/guides/mongodb/clustering/sharding.md)
  - [MongoDBOpsRequest](/docs/concepts/day-2-operations/mongodbopsrequest.md)
  - [Reconfigure Overview](/docs/guides/mongodb/reconfigure/overview.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```console
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/examples/mongodb](/docs/examples/mongodb) directory of [kubedb/docs](https://github.com/kubedb/docs) repository.

Now, we are going to deploy a  `MongoDB` sharded database using a supported version by `KubeDB` operator. Then we are going to apply `MongoDBOpsRequest` to reconfigure its configuration.

### Prepare MongoDB Shard

Now, we are going to deploy a `MongoDB` sharded database with version `3.6.8`.

### Deploy MongoDB database 

At first, we will create `mongod.conf` file containing required configuration settings.

```ini
$ cat mongod.conf
net:
   maxIncomingConnections: 10000
```
Here, `maxIncomingConnections` is set to `10000`, whereas the default value is `65536`.

Now, we will create a configMap with this configuration file.

```console
$ kubectl create configmap -n demo mg-custom-config --from-file=./mongod.conf
configmap/mg-custom-config created
```

In this section, we are going to create a MongoDB object specifying `spec.configSource` field to apply this custom configuration. Below is the YAML of the `MongoDB` CR that we are going to create,

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
      configSource:
          configMap:
            name: mg-custom-config
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: standard
```

Let's create the `MongoDB` CR we have shown above,

```console
$ kubectl create -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/reconfigure/mg-shard-config.yaml
mongodb.kubedb.com/mg-sharding created
```

Now, wait until `mg-sharding` has status `Running`. i.e,

```console
$ kubectl get mg -n demo                                                                                                                                             20:05:47
NAME          VERSION    STATUS    AGE
mg-sharding   3.6.8-v1   Running   3m23s
```

Now, we will check if the database has started with the custom configuration we have provided.

First we need to get the username and password to connect to a mongodb instance,
```console
$ kubectl get secrets -n demo mg-sharding-auth -o jsonpath='{.data.\username}' | base64 -d                                                                         11:09:51
root

$ kubectl get secrets -n demo mg-sharding-auth -o jsonpath='{.data.\password}' | base64 -d                                                                         11:10:44
xBC-EwMFivFCgUlK
```

Now let's connect to a mongodb instance and run a mongodb internal command to check the configuration we have provided.

```console
$ kubectl exec -n demo  mg-sharding-shard0-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "db._adminCommand( {getCmdLineOpts: 1})" --quiet                     21:12:47
      
  {
  	"argv" : [
  		"mongod",
  		"--dbpath=/data/db",
  		"--auth",
  		"--bind_ip=0.0.0.0",
  		"--port=27017",
  		"--shardsvr",
  		"--replSet=shard0",
  		"--clusterAuthMode=keyFile",
  		"--keyFile=/data/configdb/key.txt",
  		"--sslMode=disabled",
  		"--config=/data/configdb/mongod.conf"
  	],
  	"parsed" : {
  		"config" : "/data/configdb/mongod.conf",
  		"net" : {
  			"bindIp" : "0.0.0.0",
  			"maxIncomingConnections" : 10000,
  			"port" : 27017,
  			"ssl" : {
  				"mode" : "disabled"
  			}
  		},
  		"replication" : {
  			"replSet" : "shard0"
  		},
  		"security" : {
  			"authorization" : "enabled",
  			"clusterAuthMode" : "keyFile",
  			"keyFile" : "/data/configdb/key.txt"
  		},
  		"sharding" : {
  			"clusterRole" : "shardsvr"
  		},
  		"storage" : {
  			"dbPath" : "/data/db"
  		}
  	},
  	"ok" : 1,
  	"operationTime" : Timestamp(1598454760, 1),
  	"$gleStats" : {
  		"lastOpTime" : Timestamp(0, 0),
  		"electionId" : ObjectId("7fffffff0000000000000002")
  	},
  	"$configServerState" : {
  		"opTime" : {
  			"ts" : Timestamp(1598454762, 2),
  			"t" : NumberLong(2)
  		}
  	},
  	"$clusterTime" : {
  		"clusterTime" : Timestamp(1598454762, 2),
  		"signature" : {
  			"hash" : BinData(0,"tdnM1S4fkfbLB+1azg6xiWHeWy8="),
  			"keyId" : NumberLong("6865309861773574160")
  		}
  	}
  }
```

As we can see from the configuration of running mongodb, the value of `maxIncomingConnections` has been set to `10000`.

### Reconfigure using new ConfigMap

Now we will reconfigure this database to set `maxIncomingConnections` to `20000`. 

Now, we will edit the `mongod.conf` file containing required configuration settings.

```ini
$ cat mongod.conf
net:
   maxIncomingConnections: 20000
```

Then, we will create a new configMap with this configuration file.

```console
$ kubectl create configmap -n demo new-custom-config --from-file=./mongod.conf
configmap/mg-custom-config created
```

#### Create MongoDBOpsRequest

Now, we will use this configMap to replace the previous configMap using a `MongoDBOpsRequest` CR. The `MongoDBOpsRequest` yaml is given below,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MongoDBOpsRequest
metadata:
  name: mops-reconfigure-shard
  namespace: demo
spec:
  type: Reconfigure
  databaseRef:
    name: mg-sharding
  customConfig:
    shard:
      configMap:
        name: new-custom-config
```

Here,

- `spec.databaseRef.name` specifies that we are reconfiguring `mops-reconfigure-shard` database.
- `spec.type` specifies that we are performing `Reconfigure` on our database.
- `spec.customConfig.shard.configMap.name` specifies the name of the new configmap.

Let's create the `MongoDBOpsRequest` CR we have shown above,

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/reconfigure/mops-reconfigure-shard.yaml
mongodbopsrequest.ops.kubedb.com/ops-reconfigure-shard created
```

#### Verify the new configuration is working 

If everything goes well, `KubeDB` enterprise operator will update the `configSource` of `MongoDB` object.

Let's wait for `MongoDBOpsRequest` to be `Successful`.  Run the following command to watch `MongoDBOpsRequest` CR,

```console
$ watch kubectl get mongodbopsrequest -n demo
Every 2.0s: kubectl get mongodbopsrequest -n demo
NAME                     TYPE          STATUS       AGE
mops-reconfigure-shard   Reconfigure   Successful   3m8s
```

We can see from the above output that the `MongoDBOpsRequest` has succeeded. If we describe the `MongoDBOpsRequest` we will get an overview of the steps that were followed to reconfigure the database.

```console
$ kubectl describe mongodbopsrequest -n demo mops-reconfigure-shard  
Name:         mops-reconfigure-shard
Namespace:    demo
Labels:       <none>
Annotations:  API Version:  ops.kubedb.com/v1alpha1
Kind:         MongoDBOpsRequest
Metadata:
  Creation Timestamp:  2020-08-26T15:15:32Z
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
        f:customConfig:
          .:
          f:shard:
            .:
            f:configMap:
              .:
              f:name:
        f:databaseRef:
          .:
          f:name:
        f:type:
    Manager:      kubectl
    Operation:    Update
    Time:         2020-08-26T15:15:32Z
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
    Time:            2020-08-26T15:18:32Z
  Resource Version:  6139144
  Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/mongodbopsrequests/mops-reconfigure-shard
  UID:               8780fdc4-a7e8-4445-93fb-5d9b717a262e
Spec:
  Custom Config:
    Shard:
      Config Map:
        Name:  new-custom-config
  Database Ref:
    Name:  mg-sharding
  Type:    Reconfigure
Status:
  Conditions:
    Last Transition Time:  2020-08-26T15:15:32Z
    Message:               MongoDB ops request is being processed
    Observed Generation:   1
    Reason:                Scaling
    Status:                True
    Type:                  Scaling
    Last Transition Time:  2020-08-26T15:15:32Z
    Message:               Successfully paused mongodb: mg-sharding
    Observed Generation:   1
    Reason:                PauseDatabase
    Status:                True
    Type:                  PauseDatabase
    Last Transition Time:  2020-08-26T15:18:32Z
    Message:               Successfully Reconfigured mongodb
    Observed Generation:   1
    Reason:                ReconfigureShard
    Status:                True
    Type:                  ReconfigureShard
    Last Transition Time:  2020-08-26T15:18:32Z
    Message:               Succefully Resumed mongodb: mg-sharding
    Observed Generation:   1
    Reason:                ResumeDatabase
    Status:                True
    Type:                  ResumeDatabase
    Last Transition Time:  2020-08-26T15:18:32Z
    Message:               Successfully completed the modification process.
    Observed Generation:   1
    Reason:                Successful
    Status:                True
    Type:                  Successful
  Observed Generation:     1
  Phase:                   Successful
Events:
  Type    Reason            Age    From                        Message
  ----    ------            ----   ----                        -------
  Normal  PauseDatabase     3m42s  KubeDB Enterprise Operator  Pausing Mongodb mg-sharding in Namespace demo
  Normal  PauseDatabase     3m42s  KubeDB Enterprise Operator  Successfully Paused Mongodb mg-sharding in Namespace demo
  Normal  ReconfigureShard  42s    KubeDB Enterprise Operator  Successfully Reconfigured mongodb
  Normal  ResumeDatabase    42s    KubeDB Enterprise Operator  Resuming MongoDB
  Normal  ResumeDatabase    42s    KubeDB Enterprise Operator  Successfully Started Balancer
  Normal  Successful        42s    KubeDB Enterprise Operator  Successfully Reconfigured Database
```

Now let's connect to a mongodb instance and run a mongodb internal command to check the new configuration we have provided.

```console
$ kubectl exec -n demo  mg-sharding-shard0-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "db._adminCommand( {getCmdLineOpts: 1})" --quiet                     21:20:06
     
  {
  	"argv" : [
  		"mongod",
  		"--dbpath=/data/db",
  		"--auth",
  		"--bind_ip=0.0.0.0",
  		"--port=27017",
  		"--shardsvr",
  		"--replSet=shard0",
  		"--clusterAuthMode=keyFile",
  		"--keyFile=/data/configdb/key.txt",
  		"--sslMode=disabled",
  		"--config=/data/configdb/mongod.conf"
  	],
  	"parsed" : {
  		"config" : "/data/configdb/mongod.conf",
  		"net" : {
  			"bindIp" : "0.0.0.0",
  			"maxIncomingConnections" : 20000,
  			"port" : 27017,
  			"ssl" : {
  				"mode" : "disabled"
  			}
  		},
  		"replication" : {
  			"replSet" : "shard0"
  		},
  		"security" : {
  			"authorization" : "enabled",
  			"clusterAuthMode" : "keyFile",
  			"keyFile" : "/data/configdb/key.txt"
  		},
  		"sharding" : {
  			"clusterRole" : "shardsvr"
  		},
  		"storage" : {
  			"dbPath" : "/data/db"
  		}
  	},
  	"ok" : 1,
  	"operationTime" : Timestamp(1598455205, 1),
  	"$gleStats" : {
  		"lastOpTime" : Timestamp(0, 0),
  		"electionId" : ObjectId("7fffffff0000000000000004")
  	},
  	"$configServerState" : {
  		"opTime" : {
  			"ts" : Timestamp(1598455204, 1),
  			"t" : NumberLong(2)
  		}
  	},
  	"$clusterTime" : {
  		"clusterTime" : Timestamp(1598455205, 1),
  		"signature" : {
  			"hash" : BinData(0,"CZBx5T6rU062k9eWHd8eov6Hqb8="),
  			"keyId" : NumberLong("6865309861773574160")
  		}
  	}
  }
```

As we can see from the configuration of running mongodb, the value of `maxIncomingConnections` has been changed from `10000` to `20000`. So the reconfiguration of the database is successful.


### Reconfigure using new Data

Now we will reconfigure this database again to set `maxIncomingConnections` to `30000`. This time we won't use a new configMap. We will use the data field of the `MongoDBOpsRequest`. This will merge the new config in the existing configMap.

#### Create MongoDBOpsRequest

Now, we will use the new configuration in the `data` field in the `MongoDBOpsRequest` CR. The `MongoDBOpsRequest` yaml is given below,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MongoDBOpsRequest
metadata:
  name: mops-reconfigure-data-shard
  namespace: demo
spec:
  type: Reconfigure
  databaseRef:
    name: mg-sharding
  customConfig:
    shard:
      data:
        mongod.conf: |
          net:
            maxIncomingConnections: 30000
```

Here,

- `spec.databaseRef.name` specifies that we are reconfiguring `mops-reconfigure-data-shard` database.
- `spec.type` specifies that we are performing `Reconfigure` on our database.
- `spec.customConfig.shard.data` specifies the new configuration that will be merged in the existing configMap.

Let's create the `MongoDBOpsRequest` CR we have shown above,

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/mongodb/reconfigure/mops-reconfigure-data-shard.yaml
mongodbopsrequest.ops.kubedb.com/mops-reconfigure-data-shard created
```

#### Verify the new configuration is working 

If everything goes well, `KubeDB` enterprise operator will merge this new config with the existing configuration.

Let's wait for `MongoDBOpsRequest` to be `Successful`.  Run the following command to watch `MongoDBOpsRequest` CR,

```console
$ watch kubectl get mongodbopsrequest -n demo
Every 2.0s: kubectl get mongodbopsrequest -n demo
NAME                          TYPE          STATUS       AGE
mops-reconfigure-data-shard   Reconfigure   Successful   3m24s
```

We can see from the above output that the `MongoDBOpsRequest` has succeeded. If we describe the `MongoDBOpsRequest` we will get an overview of the steps that were followed to reconfigure the database.

```console
$ kubectl describe mongodbopsrequest -n demo mops-reconfigure-data-shard
Name:         mops-reconfigure-data-shard
Namespace:    demo
Labels:       <none>
Annotations:  API Version:  ops.kubedb.com/v1alpha1
Kind:         MongoDBOpsRequest
Metadata:
  Creation Timestamp:  2020-08-26T15:27:34Z
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
        f:customConfig:
          .:
          f:shard:
            .:
            f:data:
              .:
              f:mongod.conf:
        f:databaseRef:
          .:
          f:name:
        f:type:
    Manager:      kubectl
    Operation:    Update
    Time:         2020-08-26T15:27:34Z
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
    Time:            2020-08-26T15:30:19Z
  Resource Version:  6148406
  Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/mongodbopsrequests/mops-reconfigure-data-shard
  UID:               17889a3e-aed7-403c-9821-581a3ff5ccfd
Spec:
  Custom Config:
    Shard:
      Data:
        mongod.conf:  net:
  maxIncomingConnections: 30000

  Database Ref:
    Name:  mg-sharding
  Type:    Reconfigure
Status:
  Conditions:
    Last Transition Time:  2020-08-26T15:27:34Z
    Message:               MongoDB ops request is being processed
    Observed Generation:   1
    Reason:                Scaling
    Status:                True
    Type:                  Scaling
    Last Transition Time:  2020-08-26T15:30:19Z
    Message:               Successfully Reconfigured mongodb
    Observed Generation:   1
    Reason:                ReconfigureShard
    Status:                True
    Type:                  ReconfigureShard
    Last Transition Time:  2020-08-26T15:30:19Z
    Message:               Succefully Resumed mongodb: mg-sharding
    Observed Generation:   1
    Reason:                ResumeDatabase
    Status:                True
    Type:                  ResumeDatabase
    Last Transition Time:  2020-08-26T15:30:19Z
    Message:               Successfully completed the modification process.
    Observed Generation:   1
    Reason:                Successful
    Status:                True
    Type:                  Successful
  Observed Generation:     1
  Phase:                   Successful
Events:
  Type    Reason            Age   From                        Message
  ----    ------            ----  ----                        -------
  Normal  ReconfigureShard  59s   KubeDB Enterprise Operator  Successfully Reconfigured mongodb
  Normal  ResumeDatabase    59s   KubeDB Enterprise Operator  Resuming MongoDB
  Normal  ResumeDatabase    59s   KubeDB Enterprise Operator  Successfully Started Balancer
  Normal  Successful        59s   KubeDB Enterprise Operator  Successfully Reconfigured Database
```

Now let's connect to a mongodb instance and run a mongodb internal command to check the new configuration we have provided.

```console
$ kubectl exec -n demo  mg-sharding-shard0-0  -- mongo admin -u root -p xBC-EwMFivFCgUlK --eval "db._adminCommand( {getCmdLineOpts: 1})" --quiet
{
	"argv" : [
		"mongod",
		"--dbpath=/data/db",
		"--auth",
		"--bind_ip=0.0.0.0",
		"--port=27017",
		"--shardsvr",
		"--replSet=shard0",
		"--clusterAuthMode=keyFile",
		"--keyFile=/data/configdb/key.txt",
		"--sslMode=disabled",
		"--config=/data/configdb/mongod.conf"
	],
	"parsed" : {
		"config" : "/data/configdb/mongod.conf",
		"net" : {
			"bindIp" : "0.0.0.0",
			"maxIncomingConnections" : 30000,
			"port" : 27017,
			"ssl" : {
				"mode" : "disabled"
			}
		},
		"replication" : {
			"replSet" : "shard0"
		},
		"security" : {
			"authorization" : "enabled",
			"clusterAuthMode" : "keyFile",
			"keyFile" : "/data/configdb/key.txt"
		},
		"sharding" : {
			"clusterRole" : "shardsvr"
		},
		"storage" : {
			"dbPath" : "/data/db"
		}
	},
	"ok" : 1,
	"operationTime" : Timestamp(1598455890, 1),
	"$gleStats" : {
		"lastOpTime" : Timestamp(0, 0),
		"electionId" : ObjectId("7fffffff0000000000000006")
	},
	"$configServerState" : {
		"opTime" : {
			"ts" : Timestamp(1598455892, 1),
			"t" : NumberLong(2)
		}
	},
	"$clusterTime" : {
		"clusterTime" : Timestamp(1598455894, 1),
		"signature" : {
			"hash" : BinData(0,"nCP+UwSfVmrb2oLj0bsXM8wcuWQ="),
			"keyId" : NumberLong("6865309861773574160")
		}
	}
}
```

As we can see from the configuration of running mongodb, the value of `maxIncomingConnections` has been changed from `20000` to `30000`. So the reconfiguration of the database using the data field is successful.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```console
kubectl delete mg -n demo mg-sharding
kubectl delete mongodbopsrequest -n demo mops-reconfigure-shard mops-reconfigure-data-shard
```