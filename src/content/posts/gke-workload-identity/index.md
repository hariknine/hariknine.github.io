---
title: GKE + Workload Identity Setup
published: 2025-06-12
description: Guide to Setup Workload Identity in GKE Workloads
tags: [GKE, GCP, Kubernetes]
category: Docs
draft: false
prevTitle: "AWS IAM Roles for Service Accounts"
prevSlug: "aws-irsa"
---
Setting up Workload Identity in GKE workloads was one of the first things I had to deal with in `GKE`. I primarily worked on `EKS` until then, where we moved from using IAM users with [IAM Access Keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html) to [IRSA (IAM Roles for Service Accounts)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html). Just like AWS, at first I started with the [GCP Service Account Key](https://cloud.google.com/iam/docs/keys-create-delete), which has the same drawbacks as AWS IAM Access Keys or any similar methods i.e. the keys need to be rotated to ensure safety and even so it poses the risk of falling into the wrong hands. So we found out about Workload Identity in GKE and implemented it. This is guide on how it's done.

## GCP Service Account

We'll need a GCP Service Account with the necessary permissions. For testing we'll assign `roles/storage.viewer` for list and read access to storage buckets
```zsh
gcloud iam service-accounts create GSA_NAME --display-name "GSA for Workload Identity" --project PROJECT_ID
```
2. Assign `roles/storage.viewer` to the service account
```zsh
gcloud projects add-iam-policy-binding PROJECT_ID --member="serviceAccount:SERVICE_ACCOUNT_NAME@PROJECT_ID.iam.gserviceaccount.com" --role="roles/storage.objectViewer" --project PROJECT_ID
```
:::note
To know more about setting up `IRSA` setup, you can refer the previous article.
:::