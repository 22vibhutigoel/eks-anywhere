---
title: "Create vSphere cluster" 
linkTitle: "3. Create cluster" 
weight: 20
description: >
  Create a cluster on VMware vSphere
---

EKS Anywhere supports a VMware vSphere provider for production grade EKS Anywhere deployments.
This document walks you through setting up EKS Anywhere on vSphere in a way that:

* Deploys an initial cluster on your vSphere environment. That cluster can be used as a self-managed cluster (to run workloads) or a management cluster (to create and manage other clusters)
* Deploys zero or more workload clusters from the management cluster

If your initial cluster is a management cluster, it is intended to stay in place so you can use it later to modify, upgrade, and delete workload clusters.
Using a management cluster makes it faster to provision and delete workload clusters.
Also it lets you keep vSphere credentials for a set of clusters in one place: on the management cluster.
The alternative is to simply use your initial cluster to run workloads.
See [Cluster topologies]({{< relref "../../concepts/architecture" >}}) for details.

{{% alert title="Important" color="warning" %}}

Creating an EKS Anywhere management cluster is the recommended model.
Separating management features into a separate, persistent management cluster
provides a cleaner model for managing the lifecycle of workload clusters (to create, upgrade, and delete clusters), while workload clusters run user applications.
This approach also reduces provider permissions for workload clusters.

{{% /alert %}}

## Prerequisite Checklist

EKS Anywhere needs to:
* Be run on an Admin machine that has certain [machine
requirements]({{< relref "../install" >}}).
* Have certain
[resources from your VMware vSphere deployment]({{< relref "_index.md" >}}) available.
* Have some [preparation ]({{< relref "vsphere-preparation/_index.md" >}}) done before creating an EKS Anywhere cluster.

Also, see the [Ports and protocols]({{< relref "../ports/" >}}) page for information on ports that need to be accessible from control plane, worker, and Admin machines.

## Steps

The following steps are divided into two sections:

* Create an initial cluster (used as a management or self-managed cluster)
* Create zero or more workload clusters from the management cluster

### Create an initial cluster

Follow these steps to create an EKS Anywhere cluster that can be used either as a management cluster or as a self-managed cluster (for running workloads itself).

<!-- this content needs to be indented so the numbers are automatically incremented -->
1. Generate an initial cluster config (named `mgmt` for this example):
   ```bash
   CLUSTER_NAME=mgmt
   eksctl anywhere generate clusterconfig $CLUSTER_NAME \
      --provider vsphere > eksa-mgmt-cluster.yaml
   ```

1. Modify the initial cluster config (`eksa-mgmt-cluster.yaml`) as follows:

   * Refer to [vsphere configuration]({{< relref "./vsphere-spec/" >}}) for information on configuring this cluster config for a vSphere provider.
   * Add [Optional]({{< relref "../optional/" >}}) configuration settings as needed.
     See [Github provider]({{< relref "../optional/gitops#github-provider" >}}) to see how to identify your Git information.
   * Create at least two control plane nodes, three worker nodes, and three etcd nodes for a production cluster, to provide high availability and rolling upgrades.

1. Set Credential Environment Variables

   Before you create the initial cluster, you will need to set and export these environment variables for your vSphere user name and password.
Make sure you use single quotes around the values so that your shell does not interpret the values:
   
   ```bash
   export EKSA_VSPHERE_USERNAME='billy'
   export EKSA_VSPHERE_PASSWORD='t0p$ecret'
   ```

   {{% alert title="Note" color="primary" %}}
   If you have a username in the form of `domain_name/user_name`, you must specify it as `user_name@domain_name` to
   avoid errors in cluster creation. For example, `vsphere.local/admin` should be specified as `admin@vsphere.local`.
   {{% /alert %}}

1. Set License Environment Variable

   Add a license to any cluster for which you want to receive paid support. If you are creating a licensed cluster, set and export the license variable (see [License cluster]({{< relref "/docs/clustermgmt/support/cluster-license" >}}) if you are licensing an existing cluster):

   ```bash
   export EKSA_LICENSE='my-license-here'
   ```

   After you have created your `eksa-mgmt-cluster.yaml` and set your credential environment variables, you will be ready to create the cluster.


1. Configure Curated Packages

   The Amazon EKS Anywhere Curated Packages are only available to customers with the Amazon EKS Anywhere Enterprise Subscription. To request a free trial, talk to your Amazon representative or connect with one [here](https://aws.amazon.com/contact-us/sales-support-eks/). Cluster creation will succeed if authentication is not set up, but some warnings may be genered.  Detailed package configurations can be found [here]({{< relref "../../packages" >}}).

   If you are going to use packages, set up authentication. These credentials should have [limited capabilities]({{< relref "../../packages/prereq#setup-authentication-to-use-curated-packages" >}}):
   ```bash
   export EKSA_AWS_ACCESS_KEY_ID="your*access*id"
   export EKSA_AWS_SECRET_ACCESS_KEY="your*secret*key"
   export EKSA_AWS_REGION="us-west-2"  
   ```
     
1. Create cluster

   ```bash
   eksctl anywhere create cluster \
      # --install-packages packages.yaml \ # uncomment to install curated packages at cluster creation
      -f eksa-mgmt-cluster.yaml
   ```

1. Once the cluster is created you can use it with the generated `KUBECONFIG` file in your local directory:

   ```bash
   export KUBECONFIG=${PWD}/${CLUSTER_NAME}/${CLUSTER_NAME}-eks-a-cluster.kubeconfig
   ```
1. Check the cluster nodes:

   To check that the cluster completed, list the machines to see the control plane, etcd, and worker nodes:

   ```bash
   kubectl get machines -A
   ```

   Example command output
   ```
   NAMESPACE   NAME                PROVIDERID        PHASE    VERSION
   eksa-system mgmt-b2xyz          vsphere:/xxxxx    Running  v1.24.2-eks-1-24-5
   eksa-system mgmt-etcd-r9b42     vsphere:/xxxxx    Running  
   eksa-system mgmt-md-8-6xr-rnr   vsphere:/xxxxx    Running  v1.24.2-eks-1-24-5
   ...
   ```

   The etcd machine doesn't show the Kubernetes version because it doesn't run the kubelet service.

1. Check the initial cluster's CRD:

   To ensure you are looking at the initial cluster, list the CRD to see that the name of its management cluster is itself:

   ```bash
   kubectl get clusters mgmt -o yaml
   ```

   Example command output
   ```
   ...
   kubernetesVersion: "1.27"
   managementCluster:
     name: mgmt
   workerNodeGroupConfigurations:
   ...
   ```

   {{% alert title="Note" color="primary" %}}
   The initial cluster is now ready to deploy workload clusters.
   However, if you just want to use it to run workloads, you can deploy pod workloads directly on the initial cluster without deploying a separate workload cluster and skip the section on running separate workload clusters.
   To make sure the cluster is ready to run workloads, run the test application in the [Deploy test workload section.]({{< relref "../../workloadmgmt/test-app" >}})
   {{% /alert %}}

### Create separate workload clusters

Follow these steps if you want to use your initial cluster to create and manage separate workload clusters.

1. Generate a workload cluster config:
   ```bash
   CLUSTER_NAME=w01
   eksctl anywhere generate clusterconfig $CLUSTER_NAME \
      --provider vsphere > eksa-w01-cluster.yaml
   ```

   Refer to the initial config described earlier for the required and optional settings.

   >**NOTE**: Ensure workload cluster object names (`Cluster`, `vSphereDatacenterConfig`, `vSphereMachineConfig`, etc.) are distinct from management cluster object names.

1. Be sure to set the `managementCluster` field to identify the name of the management cluster.

   For example, the management cluster, _mgmt_ is defined for our workload cluster _w01_ as follows:

   ```yaml
   apiVersion: anywhere.eks.amazonaws.com/v1alpha1
   kind: Cluster
   metadata:
     name: w01
   spec:
     managementCluster:
       name: mgmt
   ```

1. Set License Environment Variable

   Add a license to any cluster for which you want to receive paid support. If you are creating a licensed cluster, set and export the license variable (see [License cluster]({{< relref "/docs/clustermgmt/support/cluster-license" >}}) if you are licensing an existing cluster):

   ```bash
   export EKSA_LICENSE='my-license-here'
   ```

1. Create a workload cluster in one of the following ways:
   
   * **GitOps**: See [Manage separate workload clusters with GitOps]({{< relref "../../clustermgmt/cluster-flux.md#manage-separate-workload-clusters-using-gitops" >}})
    
   * **Terraform**: See [Manage separate workload clusters with Terraform]({{< relref "../../clustermgmt/cluster-terraform.md#manage-separate-workload-clusters-using-terraform" >}})

      > **NOTE**: `spec.users[0].sshAuthorizedKeys` must be specified to SSH into your nodes when provisioning a cluster through `GitOps` or `Terraform`, as the EKS Anywhere Cluster Controller will not generate the keys like `eksctl CLI` does when the field is empty. 
   
   * **eksctl CLI**: To create a workload cluster with `eksctl`, run:
      ```bash
      eksctl anywhere create cluster \
          -f eksa-w01-cluster.yaml  \
          # --install-packages packages.yaml \ # uncomment to install curated packages at cluster creation
          --kubeconfig mgmt/mgmt-eks-a-cluster.kubeconfig
      ```
      As noted earlier, adding the `--kubeconfig` option tells `eksctl` to use the management cluster identified by that kubeconfig file to create a different workload cluster.

   * **kubectl CLI**: The cluster lifecycle feature lets you use `kubectl`, or other tools that that can talk to the Kubernetes API, to create a workload cluster. To use `kubectl`, run:
      ```bash
      kubectl apply -f eksa-w01-cluster.yaml 
      ```

1. To check the workload cluster, get the workload cluster credentials and run a [test workload:]({{< relref "../../workloadmgmt/test-app" >}})

   * If your workload cluster was created with `eksctl`, 
      change your credentials to point to the new workload cluster (for example, `w01`), then run the test application with:

      ```bash
      export CLUSTER_NAME=w01
      export KUBECONFIG=${PWD}/${CLUSTER_NAME}/${CLUSTER_NAME}-eks-a-cluster.kubeconfig
      kubectl apply -f "https://anywhere.eks.amazonaws.com/manifests/hello-eks-a.yaml"
      ```

   * If your workload cluster was created with GitOps or Terraform, the kubeconfig for your new cluster is stored as a secret on the management cluster.
     You can get credentials and run the test application as follows:
      ```bash
      kubectl get secret -n eksa-system w01-kubeconfig -o jsonpath=‘{.data.value}' | base64 —decode > w01.kubeconfig
      export KUBECONFIG=w01.kubeconfig
      kubectl apply -f "https://anywhere.eks.amazonaws.com/manifests/hello-eks-a.yaml"
      ```
1. Add more workload clusters:

   To add more workload clusters, go through the same steps for creating the initial workload, copying the config file to a new name (such as `eksa-w02-cluster.yaml`), modifying resource names, and running the create cluster command again.

## Next steps
* See the [Cluster management]({{< relref "../../clustermgmt" >}}) section for more information on common operational tasks like scaling and deleting the cluster.

* See the [Package management]({{< relref "../../packages" >}}) section for more information on post-creation curated packages installation.

## vSphere CSI Driver Deprecation

EKS Anywhere versions prior to `v0.16.0` supported the installation and management of the vSphere CSI Driver in EKS-A clusters. The vSphere CSI Driver integrates with the Cloud Native Storage (CNS) component in vCenter for the purpose of volume provisioning via vSAN, attaching and detaching the volume to VMs, mounting, formatting, and unmounting volumes from the pod within the node VM, etc. The CSI management components in EKS-A included a Kubernetes CSI controller Deployment, a node-driver-registrar DaemonSet, a default Storage Class, and a number of related Secrets and RBAC entities.

In EKS-A version `v0.16.0`, the CSI driver feature was deprecated and has been completely removed in version `v0.17.0`. However, customers can self-manage this deployment to leverage the storage options provided by vSphere in a Kubernetes-native way.

If you have created a cluster using an EKS-A version prior to `v0.16.0` and are now upgrading it using EKS-A version `v0.16.0` and above, follow the additional steps mentioned in the [vSphere upgrade steps]({{< relref "../../clustermgmt/cluster-upgrades/vsphere-and-cloudstack-upgrades#performing-a-cluster-upgrade" >}}) for proper cleanup of unmanaged vSphere CSI resources.
