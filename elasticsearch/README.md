# gnomAD Elasticsearch configuration

## Prerequisites

Before you can apply these deployments on a GKE cluster, you must have the Elastic operator installed. Theses deployments are tested with the Elastic operator version 2.9.0.

Please see the ECK documentation for details: https://www.elastic.co/guide/en/cloud-on-k8s/2.9/k8s-deploy-eck.html

Note that for production, the ECK operator is installed as part of our bootstrap configuration in ArgoCD.

## Deploying Elasticsearch

### Configurations

#### base

The base configuration is the default configuration for Elasticsearch. It is a three node cluster with 100Gi of storage for each node. This configuration is suitable for development and testing of smaller datasets. It also includes an internal load balancer for providing access to the cluster from a dataproc subnetwork.

If you don't need to make any adjustments to this configuration, you can apply it directly.

```bash
# TODO: Set your kubectl context to point to the correct cluster
kustomize build base # confirm that the configuration builds and is what you want
kustomize build base | kubectl apply -f - # apply the configuration to the cluster
```

See [custom](#custom) if you need to make adjustments to the configuration (e.g. you need to change the number of nodes, adjust disk size, or set the network CIDR allow/source range on the load balancer).


#### prod

The prod configuration patches the nodeSets to contain:

- 3 master nodes
- 3 coordinating nodes
- 4 data nodes, with 1.7Ti of storage each
- Optionally, a set of ingest nodes if you uncomment them.


It also includes a [PushSecret](https://external-secrets.io/latest/api/pushsecret/) definition to update the `gnomad-v4-elasticsearch-password` secret in the GCP Secret Manager with the current value of the `gnomad-es-elastic-user` secret that is created by the Elastic operator.

Under normal circumstances, you should not apply this configuration directly. Make The desired changes to the config in a pull request, and then merge it after review. The deployment can then be triggered from ArgoCD.

##### Common configuration changes

- To change the number of data nodes, adjust the `count` field in the `data-green` nodeSet.
- To change the disk size of the data nodes, adjust the `requests.storage` field in the `data-green` nodeSet's `volumeClaimTemplates`.
- To add an ingest node, uncomment the `ingest` nodeSet, and set the `count` to the desired number of pods. Generally, this should match the number of ingest nodes you've added to the GKE ingest node pool.


#### custom

If you need to create a custom confuguration, you start by creating a new directory in the `elasticsearch` folder, at the same level as `base` or `prod`. In this directory, you can create a new kustomization.yaml file with the `kustomize init` command.

To extend the base configuration, use `kustomize init --resources ../base`. If you need a prod-like cluster, extend the prod configuration with `kustomize init --resources ../prod`.

You can then add additional resources to the new kustomization.yaml file, or patch the existing resources as needed.

When you're ready, test that your configuration renders as expect it, and apply it with:

```bash
# TODO: Set your kubectl context to point to the correct cluster
kustomize build . # confirm that the configuration builds and is what you want
kustomize build . | kubectl apply -f - # apply the configuration to the cluster
```
