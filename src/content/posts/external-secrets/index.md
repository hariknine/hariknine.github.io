---
title: External Secrets Operator with Vault & AWS Secrets Manager
published: 2025-08-22
description: Guide to Setup Hashicorp Vault with AWS KMS or Google Cloud KMS
tags: [EKS, AWS, Kubernetes]
category: Docs
draft: true
prevTitle: "Cert-Manager + Route53 Setup"
prevSlug: "cert-manager"
---
For storing and accessing sensitive data like credentials, tokens, etc. we used the [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/managing-secrets.html) extensively whenever AWS is involved. This was in pair with the [External Secrets Operator](https://external-secrets.io), to make K8s secrets and keep them updated whenever there are changes in the copy stored in the Secrets Manager. 