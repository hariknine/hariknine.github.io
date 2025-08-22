---
title: Hashicorp Vault setup with KMS
published: 2025-08-22
description: Guide to Setup Hashicorp Vault with AWS KMS or Google Cloud KMS
tags: [EKS, AWS, GKE, GCP, Kubernetes]
category: Docs
draft: true
prevTitle: "Cert-Manager + Route53 Setup"
prevSlug: "cert-manager"
---
For storing and accessing sensitive data like credentials, tokens, etc. we used the [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/managing-secrets.html) extensively whenever AWS is involved. This was in pair with the [External Secrets Operator](https://external-secrets.io), to make K8s secrets and keep them updated whenever there are changes in the copy stored in the Secrets Manager. We also use it to store non K8s senstive data.

The primary issues with this are - cost and dependency on the AWS Secrets Manager (or any other similar cloud provider service). Here comes [Hashicorp Vault](https://developer.hashicorp.com/vault). It's a self hostable Secrets Manager of sorts that can be deployed anywhere, with support for HA and replication. 

## Vault Unsealing
1. Once Vault is deployed, it comes up in an unsealed state, which means the storage is encrypted and Vault can't access it. In K8s it is deployed as a statefulset with 3 replicas.
2. We run `vault init` inside the `vault-0` pod which generates the unseal key / root key, which is split into 5 shares (by default) and a root token (for admin access)
3. This unseal key is used to encrypt / decrypt the data encryption key, which in turn is used to encrypt the storage backend.
4. When Unsealing, Vault asks us to enter a threshold of unseal key shares (default 3/5), which is used to reconstruct the root key - which in turn is used to decrypt the data encryption key.

This process has a drawback - each replica of the Vault statefulset will have to be manually unsealed by entering the unseal key shares, each time the pods are restarted. - [Ref](https://developer.hashicorp.com/vault/docs/concepts/seal)

## Vault Auto-Unseal
To avoid this hassle, Vault has a feature called Auto Unseal. Here, we use the Cloud KMS service by the corresponding Cloud Provider to manage the encrytion / decryption of the root key. Here we'll get to know how the setup is to be done - [Ref](https://developer.hashicorp.com/vault/tutorials/auto-unseal)

## AWS KMS
We'll be using [AWS IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) for setting up AWS KMS access for Vault deployed in an EKS cluster.

1. Create AWS KMS Key
```zsh showLineNumbers=false frame=none
aws kms create-key --query "KeyMetadata.KeyId" --output text
```
2. Create AWS KMS Key alias
```zsh showLineNumbers=false frame=none
aws kms create-alias --target-key-id $KEY_ID --alias-name "alias/vault-key"
```
3. Get AWS KMS key Arn
```zsh showLineNumbers=false frame=none
export KEY_ARN=$(aws --profile sedai-labs-sandbox kms describe-key --key-id $KEY_ID --query "KeyMetadata.Arn" --output text)
```
4. Get the OIDC url from the EKS cluster
```zsh showLineNumbers=false frame=none
export OIDC_PROVIDER=$(aws eks describe-cluster --name EKS_CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f3-)
```
5. Create the IAM Trust Policy json. Replace with the correct values:
	* `ACCOUNT_ID` - AWS Account ID
```zsh showLineNumbers=false frame=none
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
						  "${OIDC_PROVIDER}:sub": "system:serviceaccount:vault:vault"
					 }
				}
		  }
	 ]
}
EOF
```
6. Create the IAM Role and note the Arn
```zsh showLineNumbers=false frame=none
export IAM_ROLE_ARN=$(aws iam create-role --role-name vault-role --assume-role-policy-document file://trust-policy.json --query "Role.Arn" --output text)
```
7. Create IAM Policy Json. This policy allows listing all the buckets in the account.
```zsh showLineNumbers=false frame=none
cat <<EOF > iam-policy.json
{
	 "Version": "2012-10-17",
	 "Statement": [
		  {
				"Sid": "VisualEditor0",
				"Effect": "Allow",
				"Action": [
					 "kms:Decrypt",
					 "kms:Encrypt",
					 "kms:DescribeKey"
				],
				"Resource": "$KEY_ARN$"
		  }
	 ]
}
EOF
```
8. Create IAM Policy and get the Arn
```zsh showLineNumbers=false frame=none
export IAM_POLICY_ARN=$(aws iam create-policy --policy-name vault-policy --policy-document file://iam-policy.json --query "Policy.Arn" --output text)
```
9. Attach the IAM Policy to the IAM Role we created 
```zsh showLineNumbers=false frame=none
aws iam attach-role-policy --policy-arn vault-policy --role-name vault-role
```

## Google Cloud KMS

## Kubernetes