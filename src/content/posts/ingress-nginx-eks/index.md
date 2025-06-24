---
title: Ingress-Nginx + EKS Setup
published: 2025-06-24
description: Guide to Setup Ingress-Nginx in EKS with ACM Certificate 
tags: [EKS, AWS, Kubernetes]
category: Docs
draft: false
prevTitle: "EKS + IRSA Setup"
prevSlug: "aws-irsa"
nextTitle: "Cert-Manager + Route53 Setup"
nextSlug: "cert-manager"
---
After setting up workloads in EKS, the requirement was to expose the workloads outside the cluster. Here we can use a [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) resource, which can be used to manage traffic to various Kubernetes Services. To manage the traffic using these ingress resources, we used [Ingress-Nginx](https://kubernetes.github.io/ingress-nginx/). It's an open source ingress controller that's widely used. Here's the guide on how to set up `ingress-nginx` in an EKS cluster and use `AWS Certificate Manager (ACM)` for the TLS termination.

## Ingress-Nginx Deployment
We can deploy `ingress-nginx` using [helm](#helm) cli or [kubectl](#kubectl) (Kubectl Manifests). First we need to get the `VPC` details and the `ACM` Arn
1. Get VPC Cidr. Replace `VPC_ID`.
```zsh showLineNumbers=false frame=none
aws ec2 describe-vpcs --vpc-ids VPC_ID --query "Vpcs[0].CidrBlock" --output text```
```
2. Get the ACM Arn. Replace `DOMAIN_NAME`
```zsh showLineNumbers=false frame=none
aws acm list-certificates --query "CertificateSummaryList[?DomainName=='DOMAIN_NAME'].CertificateArn" --output text
```

### Helm
1.  Add `ingress-nginx` helm repo
```zsh showLineNumbers=false frame=none
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```
2. Use the following custom values file. Replace `VPC_CIDR` and `ACM_ARN` from the values we obtained before
```yaml title=ingress-nginx-values.yaml
controller:
  config:
    http-snippet: |
      server {
        listen 2443;
        return 308 https://$host$request_uri;
      }
    proxy-real-ip-cidr: VPC_CIDR
    use-forwarded-headers: "true"
  service:
    enabled: true
    external:
      enabled: true
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
      service.beta.kubernetes.io/aws-load-balancer-ssl-cert: ACM_ARN
      service.beta.kubernetes.io/aws-load-balancer-ssl-ports: https
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
    type: LoadBalancer
```
3. Deploy the `ingress-nginx` chart
```zsh showLineNumbers=false frame=none
helm -n ingress-nginx upgrade --install ingress-nginx ingress-nginx/ingress-nginx -f ingress-nginx-values.yaml --create-namespace
```
### Kubectl
1. Get the `kubectl` deployment yaml for `ingress-nginx` with AWS support - [Ref](https://kubernetes.github.io/ingress-nginx/deploy/#aws)
```zsh showLineNumbers=false frame=none
curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.3/deploy/static/provider/aws/nlb-with-tls-termination/deploy.yaml
```
2. Change the values accordingly:
  * `proxy-real-ip-cidr: XXX.XXX.XXX/XX` - set value to `VPC_CIDR`
  * `service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-west-2:XXXXXXXX:certificate/XXXXXX-XXXXXXX-XXXXXXX-XXXXXXXX` - set value in `annotations` to `ACM_ARN`
3. Deploy the `kubectl` yaml
```zsh showLineNumbers=false frame=none
kubectl apply -f deploy.yaml
```

## Testing Ingress-Nginx with TLS
Since `ingress-nginx` is deployed, we'll need to wait for all the pods to come up.
```zsh showLineNumbers=false frame=none
kubectl -n ingress-nginx get pods
```
Get the `NLB` hostname that's assigned to the `ingress-nginx-controller` kubernetes service. Add this hostname as the value of an A record in your DNS provider. The A record name should be the domain where you'll need to access the kubernetes workload.
```zsh showLineNumbers=false frame=none
kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```
Get the ingress class name used by `ingress-nginx` controller. By default the value will be `nginx`
```zsh showLineNumbers=false frame=none
kubectl get ingressclasses.networking.k8s.io
```
Now we can deploy the sample workload to test this
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
spec:
  ingressClassName: nginx  # This must match the Ingress Controller's class name
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
Now try to access the domain at `https://SAMPLE_DOMAIN`. You'll see the default nginx page. You can also notice how the ssl works at `https://SAMPLE_DOMAIN`.