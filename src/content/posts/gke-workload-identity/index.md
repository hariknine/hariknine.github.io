---
title: GKE + Workload Identity Setup
published: 2025-06-12
description: Guide to Setup Workload Identity in GKE Workloads
tags: [GKE, GCP, Kubernetes]
category: Docs
draft: false
nextTitle: "EKS + IRSA Setup"
nextSlug: "aws-irsa"
---
Setting up Workload Identity in GKE workloads was one of the first things I had to deal with in `GKE`. I primarily worked on `EKS` until then, where we moved from using IAM users with [IAM Access Keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html) to [IRSA (IAM Roles for Service Accounts)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html). Just like AWS, at first I started with the [GCP Service Account Key](https://cloud.google.com/iam/docs/keys-create-delete), which has the same drawbacks as AWS IAM Access Keys or any similar methods i.e. the keys need to be rotated to ensure safety and even so it poses the risk of falling into the wrong hands. So we found out about Workload Identity in GKE and implemented it. This is guide on how it's done.

## GCP Service Account

We'll need a GCP Service Account with the necessary permissions. For testing we'll assign `roles/storage.bucketViewer` for list access to storage buckets

1. Create a GCP Service Account
```zsh showLineNumbers=false frame=none
gcloud iam service-accounts create GSA_NAME --display-name "GSA for Workload Identity" --project PROJECT_ID
```

2. Assign `roles/storage.bucketViewer` to the service account
```zsh showLineNumbers=false frame=none
gcloud projects add-iam-policy-binding PROJECT_ID --member="serviceAccount:GSA_NAME@PROJECT_ID.iam.gserviceaccount.com" --role="roles/storage.bucketViewer" --project PROJECT_ID
```

## GKE Cluster
Now that the `GSA` is ready, we'll need to make sure to enable `Workload Identity` in the GKE Cluster we have
```zsh showLineNumbers=false frame=none
gcloud container clusters update CLUSTER_NAME --location=LOCATION --workload-pool=PROJECT_ID.svc.id.goog
```
:::note
If you have a [GKE Autopilot Cluster](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview), the Workload Identity will be enabled by default with namespace - `PROJECT_ID.svc.id.goog`
:::

To enable it in an existing nodepool
```zsh showLineNumbers=false frame=none
gcloud container node-pools update NODEPOOL_NAME --cluster=CLUSTER_NAME --location=LOCATION --workload-metadata=GKE_METADATA
```
:::note
If the nodepools are created after Workload Identity is enabled in the cluster, they will also have it enabled 
:::

## GKE Workload
The GSA and GKE Cluster are setup correctly now. Next we need to let the Kubernetes workload list the storage buckets.
1. Bind the GSA to the KSA (Kubernetes Service Account)
```zsh showLineNumbers=false frame=none
gcloud iam service-accounts add-iam-policy-binding --role="roles/iam.workloadIdentityUser" --member="serviceAccount:PROJECT_ID.svc.id.goog[K8S_NAMESPACE/K8S_SA]" GSA_NAME@PROJECT_ID.iam.gserviceaccount.com --project PROJECT_ID
```
2. Create yaml for a sample kubernetes pod and a service account. The annotation `iam.gke.io/gcp-service-account: YOUR-GCP-SA@YOUR-PROJECT.iam.gserviceaccount.com` lets GKE mount a token into the pod, which lets us access the GCP resources by impersonating the account
```yaml title=workload-identity.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: K8S_SA
  namespace: K8S_NAMESPACE
  annotations:
    iam.gke.io/gcp-service-account: GSA_NAME@PROJECT_ID.iam.gserviceaccount.com
---
apiVersion: v1
kind: Pod
metadata:
  name: gcloud-cli-pod
  namespace: K8S_NAMESPACE
spec:
  serviceAccountName: K8S_SA
  containers:
  - name: gcloud-cli
    image: gcr.io/google.com/cloudsdktool/google-cloud-cli:stable
    command: ["sleep"]
    args: ["infinity"]
```
3. Use `kubectl` to create the resources
```zsh showLineNumbers=false frame=none
kubectl apply -f workload-identity.yaml
```
4. Once the pod is up, exec into the pod
```zsh showLineNumbers=false frame=none
kubectl -n K8S_NAMESPACE exec gcloud-cli-pod -it -- bash
```
5. Try to list the storage buckets in the GCP Project
```zsh showLineNumbers=false frame=none
gcloud storage buckets list
```

Et voilà! Now you can follow these steps to avoid using GSA Keys.

:::note
By default Google Cloud SDK supports this method, so any code can be run from inside the pod utilizing Workload Identity without any additional configuration
:::