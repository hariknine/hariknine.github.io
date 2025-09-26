---
title: Hashicorp Vault setup with KMS
published: 2025-09-07
description: Guide to Setup Hashicorp Vault with AWS KMS or Google Cloud KMS
tags: [EKS, AWS, GKE, GCP, Kubernetes, Vault]
category: Docs
draft: false
prevTitle: "Cert-Manager + Route53 Setup"
prevSlug: "cert-manager"
nextTitle: "External Secrets Operator with Vault & AWS Secrets Manager"
nextSlug: "external-secrets"
---
For storing and accessing sensitive data like credentials, tokens, etc. we used the [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/managing-secrets.html) extensively whenever AWS is involved. This was in pair with the [External Secrets Operator](https://external-secrets.io), to make K8s secrets and keep them updated whenever there are changes in the copy stored in the Secrets Manager. We also use it to store non K8s sensitive data.

The primary issues with this are - cost and dependency on the AWS Secrets Manager (or any other similar cloud provider service). Here comes [Hashicorp Vault](https://developer.hashicorp.com/vault). It's a self hostable Secrets Manager of sorts that can be deployed anywhere, with support for HA and replication. 

## Vault Unsealing
1. Once Vault is deployed, it comes up in a sealed state, which means the storage is encrypted and Vault can't access it. In K8s it is deployed as a statefulset with 3 replicas.
2. We run `vault init` inside the `vault-0` pod which generates the unseal key / root key, which is split into 5 shares (by default) and a root token (for admin access)
3. This unseal key is used to encrypt / decrypt the data encryption key, which in turn is used to encrypt the storage backend.
4. When Unsealing, Vault asks us to enter a threshold of unseal key shares (default 3/5), which is used to reconstruct the root key - which in turn is used to decrypt the data encryption key.

This process has a drawback - each replica of the Vault statefulset will have to be manually unsealed by entering the unseal key shares, each time the pods are restarted. - [Ref](https://developer.hashicorp.com/vault/docs/concepts/seal)

## Vault Auto-Unseal
To avoid this hassle, Vault has a feature called Auto Unseal. Here, we use the Cloud KMS service by the corresponding Cloud Provider to manage the encryption / decryption of the root key. Here we'll get to know how the setup is to be done - [Ref](https://developer.hashicorp.com/vault/tutorials/auto-unseal)

## AWS KMS
We'll be using [AWS IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) for setting up AWS KMS access for Vault deployed in an EKS cluster.

1. Create AWS KMS Key
```zsh showLineNumbers=false frame=none
export KEY_ID=$(aws kms create-key --query "KeyMetadata.KeyId" --output text)
```
2. Create AWS KMS Key alias
```zsh showLineNumbers=false frame=none
aws kms create-alias --target-key-id $KEY_ID --alias-name "alias/vault-key"
```
3. Get AWS KMS key Arn
```zsh showLineNumbers=false frame=none
export KEY_ARN=$(aws kms describe-key --key-id $KEY_ID --query "KeyMetadata.Arn" --output text)
```
4. Get the OIDC url from the EKS cluster
```zsh showLineNumbers=false frame=none
export OIDC_PROVIDER=$(aws eks describe-cluster --name EKS_CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f3-)
```
5. Create the IAM Trust Policy JSON. Replace with the correct values:
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
7. Create IAM Policy JSON. This policy allows KMS operations for the Vault auto-unseal functionality.
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
				"Resource": "$KEY_ARN"
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
10. Create Vault helm chart values file:
```yaml title=vault-values.yaml
server:
  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: >-
        IAM_ROLE_ARN
  ingress:
    enabled: true
    ingressClassName: nginx
    activeService: true
    hosts:
    - host: "VAULT_DOMAIN"
  ha:
    enabled: true
    raft:
      enabled: true
      setNodeId: true
      config: |
        cluster_name = "vault-integrated-storage"
        storage "raft" {
          path    = "/vault/data/"

          retry_join {
          leader_api_addr = "http://vault-0.vault-internal:8200"
          }
          retry_join {
          leader_api_addr = "http://vault-1.vault-internal:8200"
          }
          retry_join {
          leader_api_addr = "http://vault-2.vault-internal:8200"
          }
        }
        
        ui = true

        listener "tcp" {
          address = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_disable = "true"
        }
        service_registration "kubernetes" {}

        seal "awskms" {
          region     = "AWS_REGION"
          kms_key_id = "KEY_ID"
        }
```
11. Replace the values accordingly in the values file:
	* `IAM_ROLE_ARN` - AWS IAM role we created
	* `VAULT_DOMAIN` - Vault domain for ingress if it's needed
	* `AWS_REGION` - AWS region where the KMS is hosted 
	* `KEY_ID` - AWS KMS Key ID

## Google Cloud KMS
We'll be using [GKE Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) for setting up Google Cloud KMS access for Vault deployed in a GKE cluster.

1. Create KMS Keyring
```zsh showLineNumbers=false frame=none
gcloud kms keyrings create vault-keyring --location=LOCATION --project=PROJECT_ID
```
2. Create KMS key associated with the Keyring
```zsh showLineNumbers=false frame=none
gcloud kms keys create vault-key --location=LOCATION --purpose="encryption" --keyring=vault-keyring --project=PROJECT_ID
```
3. Create GSA (GCP Service Account)
```zsh showLineNumbers=false frame=none
gcloud iam service-accounts create vault-k8s-sa --display-name "GSA for Vault" --project PROJECT_ID
```
4. Bind the GSA to the KSA (Kubernetes Service Account)
```zsh showLineNumbers=false frame=none
gcloud iam service-accounts add-iam-policy-binding vault-k8s-sa@PROJECT_ID.iam.gserviceaccount.com --role roles/iam.workloadIdentityUser --member "serviceAccount:PROJECT_ID.svc.id.goog[vault/vault]"
```
5. Assign KMS permissions to the GSA
	* Cloud KMS CryptoKey Encrypter/Decrypter 
	```zsh showLineNumbers=false frame=none
	gcloud kms keys add-iam-policy-binding vault-key --keyring vault-keyring --location=LOCATION --member "serviceAccount:vault-k8s-sa@PROJECT_ID.iam.gserviceaccount.com" --role roles/cloudkms.cryptoKeyEncrypterDecrypter
	```
	* Cloud KMS Viewer
	```zsh showLineNumbers=false frame=none
	gcloud kms keys add-iam-policy-binding vault-key --keyring vault-keyring --location=LOCATION --member "serviceAccount:vault-k8s-sa@PROJECT_ID.iam.gserviceaccount.com" --role roles/cloudkms.viewer
	```
6. Create Vault helm chart values file:
```yaml title=vault-values.yaml
server:
  serviceAccount:
    annotations:
      iam.gke.io/gcp-service-account: >-
        vault-k8s-sa@PROJECT_ID.iam.gserviceaccount.com
  ingress:
    enabled: true
    ingressClassName: nginx
    activeService: true
    hosts:
    - host: "VAULT_DOMAIN"
  ha:
    enabled: true
    raft:
      enabled: true
      setNodeId: true
      config: |
        cluster_name = "vault-integrated-storage"
        storage "raft" {
          path    = "/vault/data/"

          retry_join {
          leader_api_addr = "http://vault-0.vault-internal:8200"
          }
          retry_join {
          leader_api_addr = "http://vault-1.vault-internal:8200"
          }
          retry_join {
          leader_api_addr = "http://vault-2.vault-internal:8200"
          }
        }

        ui = true

        listener "tcp" {
          address = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_disable = "true"
        }
        service_registration "kubernetes" {}

        seal "gcpckms" {
          project     = "PROJECT_ID"
          region      = "LOCATION"
          key_ring    = "vault-keyring" #vault-keyring
          crypto_key  = "vault-key" #vault-key
        }
```
7. Replace the values accordingly in the values file:
	* `PROJECT_ID` - GCP Project ID
	* `VAULT_DOMAIN` - Vault domain for ingress if it's needed
	* `LOCATION` - GCP location where the KMS is hosted

## Kubernetes
Now we're going to install the helm chart in the respective Kubernetes Cluster.
1. Add the helm repo
```zsh showLineNumbers=false frame=none
helm repo add hashicorp https://helm.releases.hashicorp.com
```
2. Install the helm chart using the values file we created previously
```zsh showLineNumbers=false frame=none
helm -n vault upgrade --install vault hashicorp/vault -f vault-values.yaml --create-namespace
```
3. See if the pods are present:
```zsh showLineNumbers=false frame=none
kubectl -n vault get pods
```
## Vault Cluster Setup
The deployment is done, so now we move onto the actual Vault cluster setup
1. Check the Vault status. The value of `Initialized` should be `false`
```zsh showLineNumbers=false frame=none
kubectl -n vault exec vault-0 -- vault status
```
2. Check the Vault status. The value of `Initialized` should be `false`
```zsh showLineNumbers=false frame=none
kubectl -n vault exec vault-0 -- vault status
```
3. Initialize vault at `vault-0` pod and get the cluster keys. This file will contain the `Recovery Keys` and the `Root Token`
```zsh showLineNumbers=false frame=none
kubectl -n vault exec --stdin=true --tty=true vault-0 -- vault operator init -format=json > cluster-keys.json
```
4. Check the Vault status again. The value of `Initialized` should be `true`. You can also see the other Vault pod going into `Running` state automatically. This is due to the Auto-Unsealing via the [Raft Consensus Protocol](https://developer.hashicorp.com/vault/docs/configuration/storage/raft), i.e. the corresponding Cloud KMS is used by the other pods to unseal itself.
```zsh showLineNumbers=false frame=none
kubectl -n vault exec vault-0 -- vault status
```

Et voilà! Now we have an Auto-Unsealing Vault deployment with HA. This means that the Vault stays up and in sync, regardless of restarts in any of the other replicas.

## Extras - Setting up Secrets Engine and Kubernetes Engine
To properly use Vault in Kubernetes, we're going to setup the [Vault Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2) along with [Kubernetes Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/kubernetes) to authenticate the specific KSA and then authorize using [Vault Policies](https://developer.hashicorp.com/vault/docs/concepts/policies).
1. Exec into the `vault-0` pod
```zsh showLineNumbers=false frame=none
kubectl -n vault exec -it vault-0 -- sh
```
2. Enable Vault `kv-v2` plugin
```zsh showLineNumbers=false frame=none
vault secrets enable -path=secret kv-v2
```
3. Enable Vault Kubernetes Engine
```zsh showLineNumbers=false frame=none
vault auth enable kubernetes
```
4. Configure Vault to talk to Kubernetes. `KUBERNETES_PORT_443_TCP_ADDR` is set by default by Kubernetes in its pods
```zsh showLineNumbers=false frame=none
vault write auth/kubernetes/config kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```
5. Create a Read-Only Vault Policy
```zsh showLineNumbers=false frame=none
vault policy write ro-policy - <<EOF
path "secret/*" {
  capabilities = ["read", "list"]
}
EOF
```
6. Create a Vault Kubernetes Role with the Vault Policy we created before. Replace the following:
	* `VAULT_CLIENT_KSA` - name of the KSA from which the Vault will be accessed
	* `VAULT_CLIENT_NAMESPACE` - name of the Kubernetes Namespace from which the Vault will be accessed
```zsh showLineNumbers=false frame=none
vault write auth/kubernetes/role/ro-role bound_service_account_names=VAULT_CLIENT_KSA bound_service_account_namespaces=VAULT_CLIENT_NAMESPACE policies=ro-policy ttl=24h
```
Now any pod in the `VAULT_CLIENT_NAMESPACE` namespace using the `VAULT_CLIENT_KSA` service account will be able to read any secret in the Vault. You can use clients like - [hvac](https://github.com/hvac/hvac) to access Vault using Python.
::: tip
Sample python that uses `hvac` to list the secrets in the Vault. This was run inside a Kubernetes Pod for testing the Vault setup
```py title=vault-list.py
import os
import hvac

# Get environment variables
VAULT_URL = os.getenv('VAULT_URL', 'http://vault.vault:8200')
VAULT_KUBE_ROLE = os.getenv('VAULT_KUBE_ROLE', 'ro-role')
VAULT_KUBE_JWT = os.getenv('VAULT_KUBE_JWT', '/var/run/secrets/kubernetes.io/serviceaccount/token')
VAULT_SECRETS_PATH = os.getenv('VAULT_SECRETS_PATH', 'sample')

# Create a Vault client instance
client = hvac.Client(url=VAULT_URL)

def get_vault_token():
    with open(VAULT_KUBE_JWT, 'r') as jwt_file:
        jwt = jwt_file.read()
    auth_response = client.auth.kubernetes.login(role=VAULT_KUBE_ROLE, jwt=jwt)
    client.token = auth_response['auth']['client_token']

def list_secrets():
    try:
        # List secrets at a given path
        response = client.secrets.kv.v2.list_secrets(path=VAULT_SECRETS_PATH)
        secret_list = response.get('data', {}).get('keys', [])
        return secret_list
    except hvac.exceptions.InvalidPath:
        print(f"Invalid path: {VAULT_SECRETS_PATH}")
        return []

if __name__ == "__main__":
    # Authenticate using Kubernetes JWT
    get_vault_token()

    if client.is_authenticated():
        print("Successfully authenticated with Vault.")
        secrets = list_secrets()
        if secrets:
            print("Secrets available in Vault:")
            for secret in secrets:
                print(f"- {secret}")
        else:
            print("No secrets found or unable to list secrets.")
    else:
        print("Failed to authenticate with Vault.")
```

## Extras - Using External Secrets
We can also use Vault with [External Secrets](https://external-secrets.io/latest/) to create Kubernetes Secrets with auto-updation. A guide to set this up in Kubernetes will be posted later - [here](/posts/external-secrets/).