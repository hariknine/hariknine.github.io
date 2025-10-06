---
title: GitHub actions with AWS IAM Role
published: 2025-10-06
description: Guide to access AWS resources from GitHub Actions with AWS IAM Role
tags: [GitHub, AWS, ECR]
category: Docs
draft: false
prevTitle: "External Secrets Operator with Vault & AWS Secrets Manager"
prevSlug: "external-secrets"
---
Suppose you need to access certain AWS resources from your GitHub Actions workflow. For example - you might need to build a docker image or helm chart and push to the [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html?pg=ln&sec=hs). This requires authentication to the Amazon ECR using docker / helm cli and then authorization to push the image / chart to you ECR repository. The usual method is to create an IAM user with the necessary permissions, generate Access Keys and then let the GH actions use it. This, as we know has issues of long-living credentials and risk of leaks.

AWS IAM Roles come to the rescue, again. We can create an OIDC provider in AWS which allows access from the GH actions at a repo level, in a GH Organisation or User. Let's see how it's done

# AWS Setup
We'll need to create an AWS IAM OIDC Provider with:
* Provider URL - `https://token.actions.githubusercontent.com`
* Audience - `sts.amazonaws.com`
Then we'll create the IAM role with access to the ECR
1. Create the OIDC provider
```zsh showLineNumbers=false frame=none
export OIDC_ARN=$(aws iam create-open-id-connect-provider --url https://token.actions.githubusercontent.com --client-id-list "sts.amazonaws.com" --query "OpenIDConnectProviderArn" --output text)
```
2. Create the IAM Trust Policy json. Replace with the correct values:
	* `ORG_NAME` - Organisation name or username
	* `REPO_NAME` - GitHub repo name
	* `BRANCH_NAME` - GH repo branch name
	* You can also use `*` if all branches / repos are needed
```zsh showLineNumbers=false frame=none
cat <<EOF > trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "$OIDC_ARN"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:sub": "repo:ORG_NAME/REPO_NAME:ref:refs/heads/BRANCH_NAME",
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
EOF
```
3. Create the IAM Role and get the Arn
```zsh showLineNumbers=false frame=none
export IAM_ROLE_ARN=$(aws iam create-role --role-name gh-actions-role --assume-role-policy-document file://trust-policy.json --query "Role.Arn" --output text)
```
4. Create IAM Policy Json. This policy allows pushing to all the private ECR repos in the account. Set the `Resource` accordingly if you just want to give access to specific repos.
```zsh showLineNumbers=false frame=none
cat <<EOF > iam-policy.json
{
    "Version":"2012-10-17",		 	 	 
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:CompleteLayerUpload",
                "ecr:GetAuthorizationToken",
                "ecr:UploadLayerPart",
                "ecr:InitiateLayerUpload",
                "ecr:BatchCheckLayerAvailability",
                "ecr:PutImage"
            ],
             "Resource": "*"
        }
    ]
}

EOF
```
:::warning
Avoid using wildcard resources (`*`) as this can give wide access to resources that should remain restricted. We're only using this for test purposes
:::
5. Create IAM Policy and get the Arn
```zsh showLineNumbers=false frame=none
export IAM_POLICY_ARN=$(aws iam create-policy --policy-name gh-actions-policy  --policy-document file://iam-policy.json --query "Policy.Arn" --output text)
```
6. Attach the IAM Policy to the IAM Role we created 
```zsh showLineNumbers=false frame=none
aws iam attach-role-policy --policy-arn $IAM_POLICY_ARN --role-name gh-actions-role
```

Now the IAM role can be accessed from the GH actions to push to our private ECR

# GitHub setup
Below are two sample GH workflow yamls that Logs into the ECR. Replace the values accordingly:
	* `IAM_ROLE_ARN` - value of `IAM_ROLE_ARN` we got earlier
	* `AWS_REGION` - AWS region being used
	* `ECR_REGISTRY_URL` - ECR url for helm - usually in the format - `AWS_ACCOUNT_ID.dkr.ecr.AWS_REGION.amazonaws.com`
	* `REPOSITORY_NAME` - ECR repository name for docker
1. Docker The workflow assumes the workspace contains the `Dockerfile`. 
```yaml title=docker-build.yaml
name: Push Image to ECR

on:
  workflow_dispatch:
  push:
    branches:
      - 'main'
# Required to allow GH actions to get the JWT token
permissions:
  id-token: write
jobs:
  build:
    name: Chart Push to ECR
    runs-on: self-hosted
    steps:
      - name: Checkout
        uses: actions/checkout@v5
      - name: Configure AWS Credentials for ECR
        uses: aws-actions/configure-aws-credentials@v5.0.0
        with:
          role-to-assume: IAM_ROLE_ARN
          aws-region: AWS_REGION
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      - name: Set up Docker Context for Buildx
        id: buildx-context
        run: |
          docker context create builders  
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          buildkitd-flags: --debug
          endpoint: builders
          platforms: linux/amd64,linux/arm64
      - name: Build and Push Image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          provenance: false
          push: true
          tags: ECR_REGISTRY_URL/REPOSITORY_NAME:${{ github.sha }}
```
2. Helm - The workflow assumes the workspace contains the chart. 
```yaml title=helm-package.yaml
name: Push Charts to ECR

on:
  workflow_dispatch:
  push:
    branches:
      - 'main'
# Required to allow GH actions to get the JWT token
permissions:
  id-token: write
jobs:
  build:
    name: Chart Push to ECR
    runs-on: self-hosted
    steps:
      - name: Checkout
        uses: actions/checkout@v5
      - name: Configure AWS Credentials for ECR
        uses: aws-actions/configure-aws-credentials@v5.0.0
        with:
          role-to-assume: IAM_ROLE_ARN
          aws-region: AWS_REGION
      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2
      - uses: azure/setup-helm@v4.3.0
        id: install
      - name: Get helm version
        run: |
		  echo "CHART_NAME=$(helm show chart . | grep -E "^name:" | awk '{print $2}')" >> $GITHUB_ENV
		  echo "CHART_VERSION=$(helm show chart . | grep -E "^version:" | awk '{print $2}')" >> $GITHUB_ENV
      - name: Package helm chart and push to ECR
        run: |
          helm package .
          helm push ${{ env.CHART_NAME }}-${{ env.CHART_VERSION }}.tgz oci://ECR_REGISTRY_URL
```

That's it. It's easy to setup and more secure. 

:::note
Similar methods are available for other Cloud Providers like `GCP` and `Azure`. I'll add it to this blog as I try those.
:::