# acm-sample-policies

This repo contains sample policies for Red Hat Advanced Cluster Management (ACM). These policies should be applied to the ACM hub cluster so that they can be enforced on the managed clusters.

## Considerations For Deploying Policies
These are sample policies and should only be used as references. All policies should be reviewed and updated based on individual requirements.

### Policy Placement Updates
Policies are applied using PolicySets and Placements. These use a combination of selectors to target specific clusters managed by ACM.

In order to restrict these policies from being applied to all managed clusters, they define a label to match a specific cluster name and exlude the local hub cluster. An example Placement is shown below
```
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: example-placement
spec:
  predicates:
  - requiredClusterSelector:
      labelSelector:
        matchExpressions:
        - key: vendor
          operator: In
          values:
          - OpenShift
        - key: name
          operator: NotIn
          values:
          - local-cluster
        - key: name
          operator: In
          values:
          - cluster-xyz
```

If you would like to apply these policies to your specific cluster, these Placements should be updated to reflect your cluster name or your specific requirement.

#### Placement files that need to be updated
##### PolicySets
[ocp-managed-cluster-placement](policyset/ocp-managed-clusters-policyset-placement.yaml)
[vsphere-policyset-placement](policyset/vsphere-policyset-placement.yaml)

##### Individual Policies
Additional Policies that define their own Placements
[acs-secured-cluster-configured](policies/acs-secured-cluster-configured.yaml)
[ldap-oauth-configured](policies/ldap-oauth-configured.yaml)


### Policy Updates
Example policies may require updates for requirements specific to the environment they are being deployed in. 

Below is a list of some specific considerstaions:

#### LDAP OAuth Policy
The [ldap-oauth-configured](policies/ldap-oauth-configured.yaml) policy is only provided as a template and should be updated based on specific LDAP requirements for the environment.

The policy deploys the OAuth config in the `openshift-config` namespace and creates a CronJob, and related configurations, to sync LDAP groups in the `cluster-ops` namespace. 

The `cluster-ops` namespace is used to simulate a centralized project that can be used to store all cluster admin related deployments and jobs. If this is not desired, the namespace references can be updated to an alternate namespace.

##### Configuration Considerations
- The OAuth config resource contains specific LDAP attributes that will need to be updated based on the LDAP identity provider and required LDAP attributes.
  - This configuration can be found under the `policy-oauth-config` ConfigurationPolicy
  - The OAuth configuration may need to be updated based on the Secret and ConfigMap created.
- The `ldap-config` ConfigMap contains LDAP specific configurations and attributes for an example `augmentedActiveDirectory` configuration.
  - This configuration can be found under the `policy-ldapsync-configmap` ConfigurationPolicy
  - If the LDAP group sync is being configured for an alternative schema, please reference the [OpenShift Syncing LDAP groups Documentation](https://docs.openshift.com/container-platform/4.12/authentication/ldap-syncing.html)
- The `cronjob-ldap-group-sync` CronJob is created to provide regular group syncs
  - This configuration can be found under the `policy-oauth-ldapsync-cronjob` ConfigurationPolicy
  - The CronJob configuration may need to be updated based on Secret and ConfigMap references and potential group sync command

##### Additional Considerations
Both configurations (OAuth config and group sync CronJob) require an LDAP BindPassword, and potentially certificate for Certificate Authority, to make a secure connection to the LDAP server.

This data will be stored in a Kubernetes Secret and ConfigMap in the `acm-secrets` namespace to allow for centralized management of these configurations. The policy references these configurations on the ACM Hub cluster to propage the configurations to the managed clusters.

For reference, you can find examples of these configurations in: [example-configs](example-configs/)

The `ldap-secret` Secret and `ca-config-map` ConfigMap should be created separately in the `acm-secrets` namespace prior to deploying the policy. If these resources are created with a different name, or in alternate namespaces, the policy will have to be updated to refelect those changes.


## Deploying The Policies
Policies should be applied to the ACM Hub Cluster. They can be deployed one by one manually, or deployed in bulk leveraging Kustomize

### Pre-requisites
This repo does not contain any steps to install and configure Red Hat Advanced Cluster Management (ACM). ACM should already be deployed and a MultiClusterHub instance should already be up and running.

For additional information on installing and configuring ACM, please visit:

[Red Hat Advanced Cluster Management Installation Guide](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.7/html/install/installing#doc-wrapper)

### Deployment
For ease of deployment, all directories have been setup with a kustomization.yaml file which includes the relavent files in that directory.

#### Step by Step Deployment
If you would like to deploy the files individually or in batches as per the below:

From the `acm-sample-policies` directory:
1. Deploy Namespaces - `oc apply -k ./namespaces`
2. Deploy Policies - `oc apply -k ./policies`
3. Deploy PolicySet - `oc apply -k ./policyset`

#### Deploy Everything at once
To deploy all resources in one go, you can leverage the kustomization.yaml file in the base directory.

From the `acm-sample-policies` directory:
1. Deploy all namespaces, policies, policysets - `oc apply -k .`

##### Notes
Optionally, you can review any configuration files prior to creating them using the kustomize build commands.

Examples using the `namespaces` directory, but directory can be changed as required.
- Through OC CLI - `oc kustomize ./namespaces`
- Through Kustomize binary - `kustomize build namespaces`
