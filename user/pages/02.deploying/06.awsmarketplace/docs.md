---
title: AWS Marketplace Billing
taxonomy:
    category: docs
---

***NOTE: NeuVector billing through the AWS Marketplace is scheduled to be available July 2023***

### Deploy NeuVector from AWS Marketplace Pay-As-You-Go Listing

NeuVector supports monthly billing through your AWS account in a Pay-As-You-Go (PAYG) billing subscription for SUSE support of NeuVector.

Usage is billed monthly based on the average number of nodes protected by NeuVector during the month. Please see the [NeuVector marketplace listing](https://aws.amazon.com/marketplace/pp/prodview-u2ciiono2w3h2#pdp-overview) for specific pricing tiers and other information.

NOTE: AWS Private Offers are available for NeuVector for special pricing situations in lieu of standard PAYG pricing.

### Supported Configurations
The marketplace PAYG listing supports deployment on supported target environments only. As of the July release, only EKS is supported for the billing adapter (see below options for other environments such as Rancher, Kubernetes, OpenShift etc). Each cluster can report its usage (nodes) independently, or an aggregated node count for a multi-cluster deployment can be reported. An aggregated, multi-cluster deployment can take advantage of the volume discount tiers offered by pooling all nodes across clusters into a single usage billing calculation.

NOTE: All clusters in PAYG billing (single, primary, remotes) must be running NeuVector version 5.2.0 or later.

#### Single Cluster Usage Billing
Each cluster onto which you have deployed the PAYG billing adapter through the marketplace will report usage information for your AWS account.

#### Multi-cluster Usage Billing
To be able to aggregate the node counts from multiple clusters in order to take advantage of volume discounts, the clusters must have been configured for Multi-cluster federation as described in the NeuVector [docs](https://open-docs.neuvector.com/navigation/multicluster). NeuVector on the Primary cluster MUST have been deployed through the AWS Marketplace, with the billing adapter installed in the primary cluster, in order to be able to report the primary and all downstream remote cluster node counts. Do not deploy NeuVector through the marketplace on downstream remote clusters. Use standard deployment methods (Helm, Operator, kubectl etc) described in the NeuVector [docs](https://open-docs.neuvector.com/deploying) on remote clusters.

#### Enabling PAYG Billing for Existing NeuVector Clusters

There are several options to enable PAYG billing on existing NeuVector clusters. 
- Option 1: The existing cluster must be on a supported PAYG platform. Backup the NeuVector configuration of the existing cluster, remove the NeuVector deployment, then deploy NeuVector through the AWS marketplace. After successful deployment, import the backup configuration. Note: It is recommended that the existing cluster be running version NeuVector 5.2.0 or later before the backup and removal. For Helm based deployments, this is a sample Helm upgrade command (replacing account ID, IAM role name, previous helm version values file etc):
```
helm upgrade -n neuvector neuvector  oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/suse/neuvector-csp-billing-adapter-llc/core --version 2.4.30002023052201 --create-namespace \
--set awsbilling.accountNumber=$AWS_ACCT_ID,awsbilling.roleName=$IAM_ROLE_NAME \
--set awsbilling.enabled=true,containerd.enabled=true 
-f values-x.y.z.yaml
```
- Option 2: Add the existing cluster as a federated remote cluster to a (existing or newly deployed) primary cluster which already has PAYG billing deployed on it. In this case, the existing cluster can be on any platform supported by NeuVector.

#### Enabling PAYG Billing for Rancher, OpenShift, Tanzu, or other NeuVector supported clusters

Although PAYG billing deployment is supported on a limited set of AWS platforms (only EKS at initial July release), billing for other supported NeuVector platforms can be accomplished using the multi-cluster federation configuration. As long as the primary cluster has the PAYG billing deployment of NeuVector, downstream clusters can be any supported NeuVector clusters such as Rancher, Kubernetes, OpenShift, or Tanzu. Downstream clusters can even be on-premise, or on other clouds as long as the remote cluster can be federated to the primary (with appropriate network access).

For Rancher managed downstream clusters with SSO to NeuVector, these clusters can be federated to a non-Rancher primary cluster which is deployed through the AWS marketplace in order to benefit from consolidated multi-cluster billing.


### Deploying NeuVector through the AWS Marketplace

A special billing interface is required to enable PAYG to your AWS account. This must be deployed, together with NeuVector from the AWS Marketplace listing for NeuVector. To deploy the billing adapter and NeuVector see the [deployment instructions](https://aws.amazon.com/marketplace/pp/prodview-u2ciiono2w3h2#pdp-usage) (requires AWS login).

NOTE: The NeuVector scanner image is updated daily with a new CVE database on the NeuVector docker hub registry. It is recommended that the image path be changed to allow for automated daily updates by modifying the scanner and updater image paths AFTER successful initial deployment. For example:
```
kubectl set image  deploy/neuvector-scanner-pod neuvector-scanner-pod=docker.io/neuvector/scanner:latest
kubectl set image  cronjob/neuvector-updater-pod neuvector-updater-pod=docker.io/neuvector/updater:latest
```

### Obtaining Support
Once PAYG billing is enabled for a cluster or multiple clusters, customers are eligible for support through the [SUSE Support Center](https://scc.suse.com/) (SCC) service. This is a web-based service for creating, viewing, and managing support requests. The actual link for submitting your support bundle as described below can be found [here](https://scc.suse.com/cloudsupport).

The SCC portal will require you to upload a Support Configuration bundle in order to verify your eligibility as well as provide cluster information required to start investigations. To download the support config, please go to Settings -> Configuration at the bottom of the page for the cluster in question. For multi-cluster configurations, only the Primary cluster's support config is required, even if the support inquiry is for a downstream remote cluster. If you do not have access to the Primary cluster, the remote cluster's support config is acceptable.

### Upgrading a NeuVector PAYG Cluster
The AWS marketplace PAYG listing helm chart is tied to a specific billing adapter AND NeuVector version. These are updated periodically as new versions of the billing adapter or NeuVector are released. To update the NeuVector version to the latest version supported by the marketplace listing, use the Helm update command as normal. To update the NeuVector version to a more recent version than is specified in the marketplace listing, manually change the helm values for the images (registry, paths, version tags) to point to the desired version (e.g. docker.io, neuvector/controller:5.2.5).

### Troubleshooting
This section TBD