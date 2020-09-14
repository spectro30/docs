---
title: ElasticsearchVersion
menu:
  docs_{{ .version }}:
    identifier: elasticsearh-version
    name: ElasticsearchVersion
    parent: catalog
    weight: 10
menu_name: docs_{{ .version }}
section_menu_id: concepts
---

# ElasticsearchVersion

## What is ElasticsearchVersion

`ElasticsearchVersion` is a Kubernetes `Custom Resource Definitions` (CRD). It provides a declarative way to configure and specify the docker images which are used for Elasticsearch deployment with KubeDB in a Kubernetes native way.

When you install KubeDB, an `ElasticsearchVersion` custom resource will be created automatically for every supported Elasticsearch version. You have to specify the name of `ElasticsearchVersion` crd in `spec.version` field of [Elasticsearch](/docs/concepts/databases/elasticsearch.md) crd. Then, KubeDB will use the docker images specified in the `ElasticsearchVersion` crd to create your expected database.

Using a separate crd for specifying respective docker images, and pod security policy names allow us to modify the images, and policies independent of the KubeDB operator. This will also allow the users to use a custom image for the database.

## ElasticsearchVersion Specification

As with all other Kubernetes objects, an ElasticsearchVersion needs `apiVersion`, `kind`, and `metadata` fields. It also needs a `.spec` section.

```yaml
apiVersion: catalog.kubedb.com/v1alpha1
kind: ElasticsearchVersion
metadata:
  name: 7.9.1-xpack
  labels:
    app.kubernetes.io/instance: kubedb-catalog
spec:
  authPlugin: X-Pack
  db:
    image: kubedb/elasticsearch:7.9.1-xpack
  exporter:
    image: kubedb/elasticsearch_exporter:1.1.0
  initContainer:
    image: kubedb/busybox:1.32.0
    yqImage: kubedb/elasticsearch-init:7.9-xpack
  podSecurityPolicies:
    databasePolicyName: elasticsearch-db
  version: 7.9.1
```

### metadata.name

`metadata.name` is a required field that specifies the name of the `ElasticsearchVersion` crd. You have to specify this name in `spec.version` field of [Elasticsearch](/docs/concepts/databases/elasticsearch.md) crd.

We follow this convention for naming ElasticsearchVersion crd:

- Name format: `{Original Elasticsearch version}-{security plugin name}`

- Examples: `1.9.0-opendistro`, `7.5.2-searchguard`, `7.9.1-xpack`

We use the official Elasticsearch docker images from various distributions (i.e. X-pack, SearchGuard, OpenDistro) without any modification. For avoiding the naming conflict, images are re-tagged with version and distribution name (i.e. `kubedb/elasticsearch:1.2.1-opendistro`, `kubedb/elasticsearch:7.9.1-xpack`).

### spec.version

`spec.version` is a required field that specifies the original version of the Elasticsearch database that has been used to build the docker image specified in the `spec.db.image` field.

### spec.deprecated

`spec.deprecated` is an optional field that specifies whether the docker images specified here is supported by the current KubeDB operator. For example, we have modified `kubedb/elasticsearch:6.2.4` docker image to support custom configuration and re-tagged as `kubedb/elasticsearch:6.2.4-v1`. Now, KubeDB `0.9.0-rc.0` supports providing custom configuration which required `kubedb/elasticsearch:6.2.4-v1` docker image. So, we have marked `kubedb/elasticsearch:6.2.4` as deprecated in KubeDB `0.9.0-rc.0`.

The default value of this field is `false`. If `spec.deprecated` is set `true`, the KubeDB operator will not create the database and other respective resources for this version.

### spec.db.image

`spec.db.image` is a required field that specifies the docker image which will be used to create Statefulset by KubeDB operator to create the expected Elasticsearch database.

### spec.exporter.image

`spec.exporter.image` is a required field that specifies the image which will be used to export Prometheus metrics.

### spec.initContainer.image

`spec.initContainer.image` is a required field that specifies the init container image (`busybox`) which will be used to update `sysctl` parameters.

### spec.initContainer.yqImage

`spec.initContainer.yqImage` is a required field that specifies the init container image which will be used to provide custom configuration to Elasticsearch.

### spec.podSecurityPolicies.databasePolicyName

`spec.podSecurityPolicies.databasePolicyName` is a required field that specifies the name of the pod security policy required to get the database server pod(s) running.

## Next Steps

- Learn about Elasticsearch crd [here](/docs/concepts/databases/elasticsearch.md).
- Deploy your first Elasticsearch database with KubeDB by following the guide [here](/docs/guides/elasticsearch/quickstart/quickstart.md).
