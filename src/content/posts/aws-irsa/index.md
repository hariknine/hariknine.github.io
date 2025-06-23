---
title: EKS + IRSA Setup
published: 2025-06-23
description: Guide to Setup IRSA in EKS Workloads
tags: [EKS, AWS, Kubernetes]
category: Docs
draft: false
prevTitle: "GKE + Workload Identity Setup"
prevSlug: "gke-workload-identity"
---

After getting started with `AWS` and then `EKS`, the requirement I got was to let certain workloads in the K8s cluster access certain AWS Workloads. First I created an IAM User and [IAM Access Keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html). This worked but if anyone else gets the keys somehow, they can have the same access as the IAM User the keys are associated with. This can be avoided by rotating the keys but this just delays the problem rather than avoiding it.

Turns out AWS already had a solution for this - [IRSA (IAM Roles for Service Accounts)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html). IRSA lets you bind your [IAM Role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) to your [Kubernetes Service Account](https://kubernetes.io/docs/concepts/security/service-accounts/). So, we don't need AWS IAM Users or their keys.

This is guide on how it's done.

## AWS IAM Role
1. Get the OIDC url from the EKS cluster
```zsh
export OIDC_PROVIDER=$(aws eks describe-cluster --name EKS_CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f3-)
```
:::note
Make sure there's an OIDC url associated with the EKS cluster - [Ref](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)
:::
2. Create the IAM Trust Policy json. Replace with the correct values:
	* `ACCOUNT_ID` - AWS Account ID
	* `K8S_NAMESPACE` - The Kubernetes namespace where the workload will be
	* `K8S_SA` - The Kubernetes Service Account associated with the workload
```zsh
cat <<EOF > trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/${OIDC_PROVIDER}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${OIDC_PROVIDER}:sub": "system:serviceaccount:K8S_NAMESPACE:K8S_SA"
                }
            }
        }
    ]
}
EOF
```
3. Create the IAM Role and get the Arn
```zsh
export IAM_ROLE_ARN=$(aws iam create-role --role-name IAM_ROLE_NAME --assume-role-policy-document file://trust-policy.json --query "Role.Arn" --output text)
```
4. Create IAM Policy Json. This policy allows listing all the buckets in the account.
```zsh
cat <<EOF > iam-policy-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets"
            ],
            "Resource": "*"
        }
    ]
}
EOF
```
:::warning
Avoid using broad permissions like `s3:ListAllMyBuckets` and wildcard resources (`*`) as this can give wide access to resources that should remain restricted. We're only using this for test purposes
:::
5. Create IAM Policy and get the Arn
```zsh
export IAM_POLICY_ARN=$(aws iam create-policy --policy-name IAM_POLICY_NAME  --policy-document file://iam-policy-policy.json --query "Policy.Arn" --output text)
```
6. Attach the IAM Policy to the IAM Role we created 
```zsh
aws iam attach-role-policy --policy-arn $IAM_POLICY_ARN --role-name IAM_ROLE_NAME
```

Now the IAM role allows the workloads with service account - `K8S_SA` in namespace - `K8S_NAMESPACE` in the `EKS_CLUSTER_NAME`, to list the `S3` Buckets in the AWS account. Next we'll bind this IAM Role to the actual Kubernetes workload

## EKS
1. Create yaml for a sample kubernetes pod and a service account. The annotation `eks.amazonaws.com/role-arn: IAM_ROLE_ARN` lets AWS mount a token into the pod, which lets us access the AWS resources by using the IAM Role. Replace `IAM_ROLE_ARN` in the yaml with the value from the env variable `IAM_ROLE_ARN`
```yaml title=irsa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: K8S_SA
  namespace: K8S_NAMESPACE
  annotations:
    eks.amazonaws.com/role-arn: IAM_ROLE_ARN
---
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli-pod
  namespace: K8S_NAMESPACE
spec:
  serviceAccountName: K8S_SA
  containers:
  - name: aws-cli
    image: amazon/aws-cli:2.27.40
    command: ["sleep"]
    args: ["infinity"]
```
2. Use `kubectl` to create the resources
```zsh showLineNumbers=false frame=none
kubectl apply -f irsa.yaml
```
4. Once the pod is up, exec into the pod
```zsh showLineNumbers=false frame=none
kubectl -n K8S_NAMESPACE exec aws-cli-pod -it -- bash
```
5. Try to list the S3 buckets in the AWS account
```zsh showLineNumbers=false frame=none
aws s3 ls
```

Et voilà! Now you can follow these steps to avoid using AWS IAM Access Keys.

:::note
By default AWS SDK supports this method, so any code can be run from inside the pod utilizing IRSA without any additional configuration - [Ref](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-minimum-sdk.html)
:::