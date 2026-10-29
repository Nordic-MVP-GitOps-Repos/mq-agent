## Introduction

This repo installs IBM MQ on OpenShift (OpenShift Virtualization on IBM Cloud, using ODF (Ceph RBD) block storage) using OpenShift GitOps (ArgoCD) and Kustomize. IBM Cloud Pak for Integration Platform Navigator is included for managing MQ. CP4I documentation: https://www.ibm.com/docs/en/cloud-paks/cp-integration

## Repository structure

- `argocd/` - bootstrap ArgoCD application, projects, RBAC and the operator/operand ApplicationSets
- `components/mq` - MQ operator subscription, queue manager manifests and the ODF storage / nonprod variants
- `components/platformnavigator` - Platform Navigator operator and instance
- `components/common` - IBM operator catalog source, namespace and operator group
- `components/openshift-cert-manager`, `components/cert-utils` - certificate tooling used by the queue manager TLS setup
- `components/reloader`, `components/openshift-monitoring` - Stakater Reloader (restarts queue managers on config/cert change) and user workload monitoring (MQ metrics)
- `envs/odf/nonprod` - the environment that the ApplicationSets deploy
- `examples/mq/clients` - sample MQ Java clients

## Instructions

You should have an existing OpenShift cluster available.

Make sure that the default storage class is block storage, verify that you have a storage class with the "default" annotation: ```storageclass.kubernetes.io/is-default-class: 'true'```

The MQ storage class is set in [storage-class.yaml](components/mq/variants/cloudprovider/odf/storage-class.yaml) (`ocs-storagecluster-ceph-rbd`, the ODF Ceph RBD block class). Change it if your cluster uses a different block storage class.

### Fork this repo and adapt it for your environment

First, fork this repo. Now update the following files that refer to your repo url:

- [kustomization.yaml](./argocd/kustomization.yaml)
- [bootstrap.yaml](./argocd/bootstrap.yaml)

Then check which operators and operands are selected:

- [all-operators](argocd/operators/all-operators.yaml) By default, operators are installed into the 'cp4i' namespace. If you want to change this, you can update the setting per operator.
- [all-operands.yaml](argocd/operands/all-operands.yaml) Selects which capabilities are installed (Platform Navigator, MQ, OpenShift Monitoring, Reloader).

### Install the OpenShift GitOps operator

From OperatorHub, find the OpenShift GitOps operator and install it with the recommended defaults. For details, see https://docs.openshift.com/gitops/1.12/installing_gitops/installing-openshift-gitops.html

### (Optional) Add a webhook to the OpenShift GitOps server

Add a webhook to https://your-gitops-server.com/api/webhook for the push event. For details, see https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/

### Add the bootstrap ArgoCD application to install operators

Apply the [bootstrap.yaml](./argocd/bootstrap.yaml) file to your OpenShift cluster, via the CLI or UI. The bootstrap ArgoCD application will find the [kustomization.yaml](./argocd/kustomization.yaml) file which points to the [common.yaml](./argocd/common.yaml) application and an ArgoCD ApplicationSet [all-operators.yaml](./argocd/operators/all-operators.yaml)

The "common" application creates a namespace (cp4i by default) and a catalogsource for the IBM operators.

The all-operators.yaml ApplicationSet generates ArgoCD applications for your selected operators based on a simple naming convention, pointing into the components subdirectory where Subscriptions for each operator can be found.

After applying the bootstrap file, the operators will be installed. You can check the status in the OpenShift Console under Operators / Installed Operators.

### Add the IBM entitlement key to access the container registry

Create an image pull secret in the 'cp4i' namespace with the name ibm-entitlement-key, address cp.icr.io, username cp and your entitlement key as password.

### Install operands

Now, uncomment the line referring to 'operands/all-operands.yaml' in [kustomization.yaml](./argocd/kustomization.yaml)

The [all-operands.yaml](argocd/operands/all-operands.yaml) ApplicationSet generates ArgoCD applications for Platform Navigator, MQ, OpenShift Monitoring and Stakater Reloader, using [envs/odf/nonprod](envs/odf/nonprod).

If you have enabled the webhook earlier, ArgoCD will refresh and trigger install. If you didn't, open ArgoCD and refresh the bootstrap application in the UI. This happens automatically after 3 minutes.

### Login to platform navigator

See https://www.ibm.com/docs/en/cloud-paks/cp-integration/16.1.0?topic=management-getting-initial-administrator-password

### (Optional) Enable the LetsEncrypt issuer

In the file [components/platformnavigator/base/kustomization.yaml](components/platformnavigator/base/kustomization.yaml) uncomment the line referring to 'letsencrypt-clusterissuer.yaml'
