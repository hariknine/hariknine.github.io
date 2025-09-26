---
title: External Secrets Operator with Vault & AWS Secrets Manager
published: 2025-09-26
description: Guide to Setup Hashicorp Vault with AWS KMS or Google Cloud KMS
tags: [EKS, AWS, Kubernetes, Vault]
category: Docs
draft: false
prevTitle: "Hashicorp Vault setup with KMS"
prevSlug: "vault-kms"
---
[External Secrets Operator](https://external-secrets.io) is a tool we used to create and sync Kubernetes Secrets with Secret Managers like the [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/managing-secrets.html) and [Hashicorp Vault](https://developer.hashicorp.com/vault). It lets you create Kubernetes secrets using custom resources like [ExternalSecret](https://external-secrets.io/v0.4.4/api-externalsecret/) and keeps it in sync if needed with different secret sources. Here we'll see how to set this up in a Kubernetes cluster.

## External Secrets
The external-secrets deployment and its CRDs can be deployed in our EKS cluster using Helm.
1. Add helm chart repo
```zsh showLineNumbers=false frame=none
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
```
2. Install the helm chart
```zsh showLineNumbers=false frame=none
helm -n external-secrets install external-secrets external-secrets/external-secrets --create-namespace --set installCRDs=true
```
3. Make sure the pods are running once the helm deployment is done
```zsh showLineNumbers=false frame=none
kubectl -n external-secrets get pods
```

## AWS Secrets Manager
We will be using [AWS IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) for setting up AWS Secrets Manager access for the External Secrets Operator deployed in an EKS cluster.

### IRSA
1. Get the OIDC url from the EKS cluster
```zsh showLineNumbers=false frame=none
export OIDC_PROVIDER=$(aws eks describe-cluster --name EKS_CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f3-)
```
2. Create the IAM Trust Policy JSON. Replace with the correct values:
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
						  "${OIDC_PROVIDER}:sub": "system:serviceaccount:external-secrets:aws-secrets-sa"
					 }
				}
		  }
	 ]
}
EOF
```
3. Create the IAM Role and note the Arn
```zsh showLineNumbers=false frame=none
export IAM_ROLE_ARN=$(aws iam create-role --role-name external-secrets-role --assume-role-policy-document file://trust-policy.json --query "Role.Arn" --output text)
```
4. Create IAM Policy JSON. This policy allows the external-secrets operator to get the secret value from AWS Secrets Manager
```zsh showLineNumbers=false frame=none
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetResourcePolicy",
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret",
        "secretsmanager:ListSecretVersionIds"
      ],
      "Resource": "*"
    }
  ]
}
```
5. Create IAM Policy and get the Arn
```zsh showLineNumbers=false frame=none
export IAM_POLICY_ARN=$(aws iam create-policy --policy-name external-secrets-policy --policy-document file://iam-policy.json --query "Policy.Arn" --output text)
```
6. Attach the IAM Policy to the IAM Role we created 
```zsh showLineNumbers=false frame=none
aws iam attach-role-policy --policy-arn external-secrets-policy --role-name external-secrets-role
```

### Kubernetes Setup
Now we'll need to create 2 custom k8s resources - [ClusterSecretStore](https://external-secrets.io/v0.5.1/api-clustersecretstore/) and [ExternalSecret](https://external-secrets.io/v0.5.1/api-externalsecret/). `ExternalSecret` is the resource that creates the k8s secret by accessing the AWS Secrets Manager / Vault using the `ClusterSecretStore`. `ClusterSecretStore` is a cluster-wide resource and the `ExternalSecret` is a namespaced resource.

1. Create the `ClusterSecretStore` resource. The yaml below also consists of the k8s service account that has access to the IAM Role we created earlier. Replace `IAM_ROLE_ARN` with the value we obtained earlier.
```yaml title=clustersecretstore.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: IAM_ROLE_ARN
  name: aws-secrets-sa
  namespace: external-secrets
---
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: aws-secrets-sa
            namespace: external-secrets
```
2. Create the `ExternalSecret` resource. This creates a k8s secret called `ExternalSecret` in the namespace `default`, with values from the AWS Secret `aws-secret`. This is set to check and refresh the values of the k8s secret every 3 minutes. So any changes made in the AWS Secrets Manager will be reflected in the cluster secret too.
```yaml title=externalsecret.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: aws-secret
  namespace: default
spec:
  refreshInterval: 3m0s
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets   
  dataFrom:
  - extract:
      key: aws-secret
  target:
    creationPolicy: Owner
    name: aws-secret
```
3. The k8s secret with the name should be available
```zsh showLineNumbers=false frame=none
kubectl -n default describe secrets aws-secret
```

## Vault
To setup access to the Vault server from the External Secrets Operator, we will be using the Kubernetes Auth method - [Ref](/posts/vault-kms/#extras---setting-up-secrets-engine-and-kubernetes-engine)

### Vault Kubernetes Auth
We will be creating a `Vault Policy`, `Vault Kubernetes Role` with access to the Secrets stored in the Vault server assigned to the Kubernetes Service Account used by the External Secrets Operator.
1. Exec into the `vault-0` pod
```zsh showLineNumbers=false frame=none
kubectl -n vault exec -it vault-0 -- sh
```
2. Login to the vault server
```zsh showLineNumbers=false frame=none
vault login
```
3. Create a Vault policy with access to the Vault secrets from which we want to create k8s secrets.
```zsh showLineNumbers=false frame=none
vault policy write external-secrets-policy - <<EOF
path "vault-secrets/*" {
  capabilities = ["read", "list"]
}
EOF
```
4. Create a Vault Kubernetes Role that binds the above policy to the Kubernetes Service Account used by the External Secrets Operator.
```zsh showLineNumbers=false frame=none
vault write auth/kubernetes/role/external-secrets-role bound_service_account_names=external-secrets bound_service_account_namespaces=external-secrets policies=external-secrets-policy ttl=24h
```

### Kubernetes Setup
Now we'll need to create the `ClusterSecretStore` and `ExternalSecret` resources for Vault.
1. Create the `ClusterSecretStore` resource.
```yaml title=clustersecretstore.yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: hashicorp-vault
spec:
  provider:
    vault:
      auth:
        kubernetes:
          mountPath: kubernetes
          role: external-secrets-role 
      path: vault-secrets
      server: http://vault.vault:8200
      version: v2
```

2. Create the `ExternalSecret` resource.
```yaml title=clustersecretstore.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: vault-secret
  namespace: default
spec:
  dataFrom:
  - extract:
      conversionStrategy: Default
      decodingStrategy: None
      key: vault-secrets/data/vault-secret-name
      metadataPolicy: None
  refreshInterval: 2m0s
  secretStoreRef:
    kind: ClusterSecretStore
    name: hashicorp-vault
  target:
    creationPolicy: Owner
    deletionPolicy: Retain
    name: vault-secret
```
3. The k8s secret with the name should be available
```zsh showLineNumbers=false frame=none
kubectl -n default describe secrets vault-secret
```

And there you have it! We have successfully set up the External Secrets Operator with AWS Secrets Manager and HashiCorp Vault.