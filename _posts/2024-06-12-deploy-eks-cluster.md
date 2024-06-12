---
title: Deploy EKS cluster with Terraform
categories: [k8s]
tags: [eks,k8s,aws, terraform]
---
The aim of this terraform project to is to create a ready to cluster with all the necessary addons and features enabled.

All the source is available here: [https://github.com/rizvn/eks-terraform](https://github.com/rizvn/eks-terraform)

The following tapology will be deployed:

![alt text](/assets/img/eks-terraform.svg "eks-terraform")

- VPC with 
  - 3 public subnets
  - 3 private subnets
  - NAT Gateway
  

-  EKS cluster with
   - OIDC enabled for IRSA
   - With Addons
     - CoreDNS
     - VPC CNI with Network policy support
     - Kube-proxy

- Ingress-only nodegroup 
- General nodegroup



Additional modules are defined under the extras and can enabled through values in `01-variables.tf`

Additional modules include:
- AWS Load balancer
- Cluster Autoscaler
- Karpenter Autoscaler
- Nginx Ingress (internal and external)
- Users (IAM users with EKS Access)
- EFS fs connected to the EKS cluster using EFS CSI driver


Clone the git repo
```bash 
git clone git@github.com:rizvn/eks-terraform.git

cd eks-terraform
```

Set Default AWS Profile to use. This should be the profile that has the necessary permissions to create the resources in the account
```bash
export AWS_PROFILE=test
```

Update values and flags in `01-variable.tf`

Deploy 
```bash
terraform init
terraform apply
```


Update local kubeconfig for the new cluster
```bash
aws eks update-kubeconfig --region <your-region> --name  <your-cluster-name>
```

List nodes to test connectivity
```bash
kubectl get nodes
````
