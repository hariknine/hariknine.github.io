---
title: Cert-Manager + Route53 Setup
published: 2025-08-19
description: Guide to Setup Cert-Manager with DNS01 verification in Route53
tags: [AWS, Kubernetes]
category: Docs
draft: false
prevTitle: "Ingress-Nginx + EKS Setup"
prevSlug: "aws-irsa"
nextTitle: "Hashicorp Vault setup with KMS"
nextSlug: "vault-kms"
---
After setting up `ingress-nginx` with an ACM certificate in EKS clusters, we also needed to set up `ingress-nginx` in `GKE` and `AKS` clusters, which was easy since we just needed to use the default `ingress-nginx` helm chart to deploy to whichever cluster as required. But TLS was an issue.

Solution - since the domain is managed in `Amazon Route 53`, we can use `cert-manager` with `DNS01` challenge to get TLS working along with Route53 in any cluster, regardless of the cloud provider. Here we'll see the steps required for this setup. DNS01 challenge essentially involves you proving you own the domain by creating a TXT record, which is then checked by the CA (here it will be [Let's Encrypt](https://letsencrypt.org/)) and once verified you're given the TLS cert. For more details - [Ref](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge)

## AWS setup
1. Create the IAM user to be used by `cert-manager`
```zsh showLineNumbers=false frame=none
aws iam create-user --user-name route53-certmanager-user
```
2. Use the following policy json that allows `cert-manager` to access the route53 zone
```yaml title=route53-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "arn:aws:route53:::change/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/*",
      "Condition": {
        "ForAllValues:StringEquals": {
          "route53:ChangeResourceRecordSetsRecordTypes": ["TXT"]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": "route53:ListHostedZonesByName",
      "Resource": "*"
    }
  ]
}
```
3. Create and attach policy to the IAM User
```zsh showLineNumbers=false frame=none
aws iam put-user-policy --user-name route53-certmanager-user --policy-name route53-certmanager-policy --policy-document file://cert-manager/route53-policy.json
```
4. Create IAM Secret keys for the IAM user. Note the fields - `AccessKeyId` and `SecretAccessKey`
```zsh showLineNumbers=false frame=none
aws iam create-access-key --user-name route53-certmanager-user
```

## Cert-Manager setup
For ease of management let's use `helm` to deploy `cert-manager`. 
1. Run the following command to deploy it in your K8s cluster:
```zsh showLineNumbers=false frame=none
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.18.2 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```
2. Replace `AWS_ACCESS_KEY_ID_BASE64` and `AWS_SECRET_ACCESS_KEY_BASE64` in the following K8s secrets yaml with the base64 encoded values of `AccessKeyId` and `SecretAccessKey` we created earlier. Note that the secret should be created in the same namespace where `cert-manager` is deployed - here it's `cert-manager`.
```yaml title=route53-credentials-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: route53-credentials-secret
  namespace: cert-manager
data:
  aws-access-key-id: AWS_ACCESS_KEY_ID_BASE64
  aws-secret-access-key: AWS_SECRET_ACCESS_KEY_BASE64
type: Opaque
```
3. Create the secret
```zsh showLineNumbers=false frame=none
kubectl apply -f route53-credentials-secret.yaml
```
4. Replace `ROUTE53_DOMAIN_NAME` with the Route53 domain name. Example: `yourdomain.com`
```yaml title=clusterissuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    privateKeySecretRef:
      name: letsencrypt-prod
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - dns01:
        route53:
          accessKeyIDSecretRef:
            key: aws-access-key-id
            name: route53-credentials-secret
          region: us-east-1
          secretAccessKeySecretRef:
            key: aws-secret-access-key
            name: route53-credentials-secret
      selector:
        dnsZones:
        - ROUTE53_DOMAIN_NAME
```
5. Create the ClusterIssuer. This will be used in our ingress resource to request the TLS certificate.
```zsh showLineNumbers=false frame=none
kubectl apply -f clusterissuer.yaml
```

## Testing Ingress-Nginx with TLS
1. Now we can deploy the sample workload to test this. Replace `SAMPLE_DOMAIN` with your domain as required
```yaml title=nginx-deployment.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx  # This must match the Ingress Controller's class name
  tls:
  - hosts:
    - SAMPLE_DOMAIN
    secretName: domain-cert-secret
  rules:
    - host: SAMPLE_DOMAIN  # Replace with your actual domain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```
2. Get the `ingress-nginx` loadbalancer IP (if applicable). This is usually applicable in most clusters.
```zsh showLineNumbers=false frame=none
kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
3. `OR` - Get the `ingress-nginx` loadbalancer hostname (if applicable). This is usually applicable in EKS.
```zsh showLineNumbers=false frame=none
kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```
4. Create Route53 record for `SAMPLE_DOMAIN` with the value as either the loadbalancer IP or loadbalancer hostname.
5. Wait for a few seconds for the DNS01 challenge and DNS propagation to be over.
6. Try to access `https://SAMPLE_DOMAIN`. Et voilà! The endpoint is accessible with working TLS
