
# Kubernetes Game Deployment on Amazon EKS

This repository provides an end-to-end guide for deploying a game application on an Amazon Elastic Kubernetes Service (EKS) cluster. The deployment includes setting up the EKS cluster, deploying the game application, configuring networking, and managing security and scaling.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Setup EKS Cluster](#setup-eks-cluster)
- [Deploy the Game Application](#deploy-the-game-application)
- [Iam roles,policy and Ingress controller](#networking-and-ingress)

## Prerequisites

Before you begin, ensure you have the following:

- An AWS account
- AWS CLI installed and configured
- `eksctl` CLI installed
- `kubectl` CLI installed
- Docker installed

## Architecture

The architecture for this deployment includes:

- **EKS Cluster**: Managed Kubernetes cluster on AWS.
- **Game Application**: A containerized game application deployed on the EKS cluster.
- **Load Balancer**: AWS Load Balancer to distribute traffic to the application.
- **Ingress Controller**: To manage external access to the application.

## Setup EKS Cluster

  **Create an EKS Cluster**

    Use `eksctl` to create a new EKS cluster.

    ```sh
    eksctl create cluster --name game-cluster --region us-west-2 --fargate
    ```

## Deploy the Game Application

1. **Create Kubernetes Deployment, Namespace, ingress and Service**

 ```
---
apiVersion: v1
kind: Namespace
metadata:
  name: game-2048
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: game-2048
  name: deployment-2048
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048
  replicas: 5
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048
    spec:
      containers:
      - image: public.ecr.aws/l6m2t8p7/docker-2048:latest
        imagePullPolicy: Always
        name: app-2048
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  namespace: game-2048
  name: service-2048
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    app.kubernetes.io/name: app-2048
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: service-2048
              port:
                number: 80

```

Apply the manifests:
    
    ```sh
    kubectl apply -f game-deployments.yaml
    ```

## Networking and Ingress

1. **Install an Ingress Controller**

  Add helm repo

```
helm repo add eks https://aws.github.io/eks-charts
```

Update the repo

```
helm repo update eks
```

Install

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```

Verify that the deployments are running.

```sh
kubectl get deployment -n kube-system aws-load-balancer-controller
```

 ## Security

1. **Configure IAM Roles for Service Accounts (IRSA)**

    Associate an OIDC provider with your EKS cluster and create IAM roles for service accounts.

    ```sh
    eksctl utils associate-iam-oidc-provider --cluster game-cluster --approve
    ```

    Create IAM roles and policies as needed and annotate the Kubernetes service account.
    ```sh
    aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
    ```

    ```sh
    eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
    ```

## Cleanup

To clean up the resources created in this guide, follow these steps:

1. **Delete Kubernetes Resources**

    ```sh
    kubectl delete -f deployments.yaml
 
    ```

2. **Delete EKS Cluster**

    ```sh
    eksctl delete cluster --name game-cluster
    ```
