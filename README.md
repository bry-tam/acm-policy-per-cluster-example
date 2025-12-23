###  ACM Policy Management Per Policy / Per Cluster

#### Description
This repository represents an example of how to structure a git source repository such that each policy can be versioned independently on the hub.

The output on the ACM hub will have each cluster in their own `ClusterSet` and all the policies for that cluster in a common `policies-MANAGEDCLUSTER` namespace.

For an indepth view and explanation of the kustomize output, view the [kustomize-flow.md](docs/kustomize-flow.md)

### Directory Structure
Each cluster is represented with a directory in the `clusters` root directory.  The cluster specific details is in a "cluster" subdirectory, this is done to satisfy kustomize circular references.  The "cluster" subdirectory is referenced from every policy kustomize path, this is done to update `Placements`, `ClusterSets`, `ManagedClusterSetBinding` are all created specific for this cluster.

The `PolicySet` SuffixTransformer is highly recommended to make identifying any `PolicySets` on the hub easier to identify.

Under each `clusters/MANAGEDCLUSTER/policies` directory are subdirectories representing each policy that should be applied to that cluster.  The `kustomization.yaml` will include a reference to the `../../cluster` directory along with the resources to the `/policies/POLICY`.  This directory is the entry point for the Argo Application that will create the Policy and associated data on the hub.
```
clusters/ocphbprd/
├── cluster
│   ├── kustomization.yaml
│   ├── managedcluster.yml
│   └── policyset-suffixer.yml
└── policies
    ├── acm_ensure-placement-toleration
    │   └── kustomization.yaml

```

The Policies are all maintained in subdirectories under the "policies" directory.  The "common" subdirectory provides the required `namespace`, `ManagedClusterSetBinding`, `ManagedClusterSet` and cluster specific `Placement`.  The common subdirectory should be included in the kustomize chain from the `clusters/MANAGEDCLUSTER/cluster/kustomization.yaml` 
```
policies/
├── acm
│   ├── ensure-placement-toleration
│   ├── feature-flags-placement
│   ├── klusterlet-infra
│   ├── kubeadmin-config-trustca
│   └── managedserviceaccount
├── common
│   ├── acm-placements
│   ├── kustomization.yaml
│   └── managedcluster
└── ocp
    ├── cluster-configs
    ├── cluster-health
    └── operators
```

ArgoCD should apply the policies to the hub via the "release-management" directory.  This could be managed in a separate repository if desired.  This is where cluster, policy and revision relationship is managed.

The base/appset.yml contains the basic details for the generated Argo Application.  This allows a single place to manage how all Argo Applications are generated.  Note the kustomization is of kind `Component`, this is to ensure the full resource tree is available when it is called.  Also the appset.yml is a patch file, thus allowing the name of the `ApplicationSet` to be specified in the policies directories creating an ApplicationSet for each Policy.

The Argo Application created as the root app will have a `path: release-management/` - this application should have a revision of "main" to ensure all applications are targetting the expected revision.
```
release-management/
├── base
│   ├── appset.yml
│   └── kustomization.yml
├── kustomization.yaml
└── policies
    ├── acm_feature-flags-placement
    │   ├── kustomization.yml
    │   └── policy-appset.yml
    ├── acm_klusterlet-infra
    │   ├── kustomization.yml
    │   └── policy-appset.yml
```

The `policies/POLICY/policy-appset.yml` defines the generator portion of the `ApplicationSet`.  The use of a matrix generator allows combining the elements between the two lists together.  This allows a global value to be set for "policy" and "revision" for use the `ApplicationSet` templating, and more importantly to allow the revision to be overridden on a pre-cluster bases.  Notice in the example below the revision on cluster "ocpbd" is overridden from the global value.

You can also override the template section of ApplicationSet if required.

```
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: ocp_operator_gitops
  namespace: openshift-gitops
spec:
  generators:
  - matrix:
      generators:
        - list:
            elements:
              - policy: ocp_operator_gitops
                revision: main
        - list:
            elements:
            - cluster: ocpac

            - cluster: ocpbd
              revision: acm_20251120.0

            - cluster: ocpce

            - cluster: ocphbprd
```

#### Implementation Details
- **Placements** - Placements in the `PolicyGenerator` can reference the named "per-cluster-placement" since this will be updated specific for the cluster associated with that namespace.  If you don't want a policy to apply to a specific cluster, just remove it from the
- **Polices** - for a Policy to be applied to a cluster the following must be satisfied
  1. The `ApplicationSet` needs to be created for the Policy and included in the root `release-management/kustomization.yaml` file.
  2. The ApplicationSet needs to list the cluster in the generator.
  3. The kustomize overlay needs to be created for the cluster (and any cluster that the policy should be applied to) in the `/clusters/CLUSTER/policies/POLICY`.
- **Clusters** - Each cluster needs to be listed in `release-management/clusters/cluster-appset.yml`.  This application will apply the various objects tying the managed cluster with the policy.  This includes the `ManagedCluster` to set the cluster-set annotation, `ManagedClusterSet` to create the per-cluster cluster set.  The `Placement` specific to that cluster/clusterset.  And the `ManagedClusterSetBinding` to allow the policies in the `policies-CLUSTER` namespace to be references in the Placement.


Notes:
  - The policies in this repo are just examples to showcase how the entire layout works.  They should be replace with updated policies.
  - The amount of kustomize files/directories required to be managed is fairly large using a layout like this.  But this give the highest flexability for managing ACM policies.
  - There are a number of items created repeatedly for each policy/cluster combo.  Things like the `Placement` and `MangedCluster`.  These are blocked from being applied by the policy, instead they are applied by a separate AppSet/Application that only applies the cluster specific objects (`Placement`, `ManagedCluster`, `ManagedClusterSet`, `ManagedClusterSetBinding`)
