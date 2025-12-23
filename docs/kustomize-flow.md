## Understanding how the kustomize documents are implemented and how the Argo Applications relate

To better understand how to implement this repository structure this document will explain the three directories and how the relate.  The goal of each section is to provide a visual and texual understand of what is being created as the output from running `kustomize build` command.

1. [./release-management](#release-management)
2. [clusters/**/cluster](#clusters)
3. [clusters/**/policies/## --> policies/##](#policy-per-cluster)

Within the diagrams below, the blocks marked "**_kustomize entry point_**" are representative of the `kustomization.yaml` files within the example directories.

### Release Management
![release-management kustomize structure](images/release-management.jpg)
The `release-management` directory is the root entry path.  An Argo Application path to this directory will create the rest of the `ApplicationSets` and `Applications`.  This could be managed in a separate repository instead of within the same repo as the clusters/policies.

An `AppProject` is defined for use by the [Policy per Cluster](#policy-per-cluster) to limit the ability to create/apply the cluster specific objects for those applications.

Running `kustomize build` command at this level will produce an Argo `ApplicationSet` to deploy the [cluster](#clusters) specific data and an `ApplicationSet` for each [Policy](#policy-per-cluster). Along with the `AppProject` for deploying the Policies per cluster.

You can control the version of the repo deployed by any of the Argo `Applications` in this section by setting the  `revsion: TAG` to either a tag or branch in the repo.  General usage would expect the `./clusters` to use the HEAD/main tag to always have the latest, but this is not a requirement.

For versioning of Policies you can specify a default in the first list that includes the policy details, and override this for each cluster as needed.  In this example there are four clusters that will receive the ocp_cluster-health policy.  The "**_ocpac_**" cluster will be deployed with revision "20251225-1", the other clusters will deploy the latest on the main branch. 
```yaml
spec:
  generators:
  - matrix:
      generators:
        - list:
            elements:
              - policy: ocp_cluster-health
                revision: main
        - list:
            elements:
            - cluster: ocpac
              revision: 20251225-1

            - cluster: ocpbd

            - cluster: ocpce

            - cluster: ocphbprd
```

### Clusters
![cluster kustomize structure](images/clusters.jpg)
The `ApplicationSet` created from `/release-management/clusters` will generate an Argo `Application` for each cluster listed in the generator.

The cluster specific details specified in the `kustomization.yaml` is also used by the [Policies per Cluster](#policy-per-cluster), however they are only applied to the ACM Hub cluster from this `Application` and not with the policies.  

The `Namespace` and `Placement` created will apply to all Policies which target this cluster.

Because we are creating the `ManagedCluster`, `ManagedClusterSet`, `Placement`, `Namespace`, and `ManagedClusterSetBinding` we can use kustomize to correct any references between these objects automating all of the ACM configuration that is needed.  Since we are using a 1:1 relationship between the `ManagedClusters` and `ManagedClusterSets` we don't need to worry about creating multiple placements specific to different clusters.

### Policy per Cluster
![policy per cluster kustomize structure](images/cluster-policy.jpg)

Each Policy being managed will have it's own Argo `ApplicationSet` which will create an Argo `Application` for each it is being deployed to.  As noted in the [Release Management](#release-management) section each cluster can receive a different version of the Policy.  

This versioning scheme allows finite control over when and where Policies change, at the expense of more time and effort required to manage.

  > **_Note:_** The kustomize references the [Cluster](#clusters) component, this is only for purposes of setting cluster specific details on the Policy output. The cluster objects are not created/managed by Argo in this Application.  The `AppProject` specifically denies the ability.
  >
  > The Argo `Application` also includes hooks to prevent these items from blocking sync waves

In the `PolicyGenerator` each `Placement` specification should use `placementName: per-cluster-placement`.  Below shows how to set this for the `PolicySet`, the yaml configuration is the same for Policies generated without a `PolicySet`

  > **_Note:_** Use of `PolicySets` is optional.  I prefer it as a default, but I'm me -- you should not feel it is correct one way or the other.

```yaml
policySets:
  - name: cluster-health
    placement:
      placementName: per-cluster-placement
```

## A Note about `kind: Kustumization` vs. `kind: Component`

All "**_kustomization.yaml_**" related to Policies and [Release Management](#release-management) _(except the base directory)_ are `kind: Kustomization`.

All "**_kustomization.yaml_**" related to Clusters, `release-management/base` and kustomize configurations/transformers are `kind: Component`.

This has to do with when they are processed by the kustomize build process and how the object tree is passed.

To understand the concept of components reading the [Kubernetes Enhancement Proposal](https://github.com/kubernetes/enhancements/tree/master/keps/sig-cli/1802-kustomize-components) for the feature.
