---
title: Create an EKS cluster with eksctl and aws cli
categories: [k8s]
tags: [eks,k8s,aws,eksctl]
---

# Create an EKS cluster with eksctl and aws cli

### Install aws cli 
Install and aws cli from [here](https://aws.amazon.com/cli/)

### Install eksctl
Install eksctl on you machine [here](https://eksctl.io/)

### Create a clustter config
The cluster config file creates the eks cluster 

**File:** ```01-cluster-config.yaml```

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-eks-cluster
  region: eu-west-2

nodeGroups:
  - name: workers
    desiredCapacity: 3
    instancesDistribution:
      maxPrice: 0.085 # Set an appropriate max price for Spot instances
    availabilityZones: ["eu-west-2a", "eu-west-2b", "eu-west-2c"]
    privateNetworking: true # If you want to use private networking for the nodes
    iam:
      attachPolicyARNs:
        - arn:aws:iam:YOUR-ACCOUNT-NUMBER:aws:policy/workers-policy

    # insatnce types for the nodes
    instanceType:
      - m5.large
      - c5.large


    # Specify subnets for each Availability Zone
    subnets:
      - subnet-0-az-a
      - subnet-1-az-b
      - subnet-2-az-c

vpc:
  # create vpc with this cidr
  cidr: 10.0.0.0/16

  subnets:
    private:
      eu-west-2a:
        id: subnet-0-az-a
        cidr: 10.0.1.0/24
      eu-west-2b:
        id: subnet-1-az-b
        cidr: 10.0.2.0/24
      eu-west-2c:
        id: subnet-2-az-c
        cidr: 10.0.3.0/24
    public:
      eu-west-2a:
        id: subnet-public-0-az-a
        natGateway: true
        cidr: 10.0.4.0/24
      eu-west-2b:
        id: subnet-public-1-az-b
        natGateway: true
        cidr: 10.0.5.0/24
      eu-west-2c:
        id: subnet-public-2-az-c
        natGateway: true
        cidr: 10.0.6.0/24
```

### Update the account in the cluster config
Set the account number in the cluster config file. Change **123** in the command below to your actual aws acccount number
```bash
sed -i 's/YOUR-ACCOUNT-NUMBER/123/g' 01-cluster-config.yaml
```

### Create worker policy
Create policy for worker nodes in the cluster to able to pull images from your private ecr repository 

**File:**  ```02-worker-iam-policy.json```
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ],
      "Resource": "*"
    }
  ]
}
```

### Deploy the policy
Run the following command in terminal
```bash
aws iam create-policy --policy-name worker-policy --policy-document 02-worker-iam-policy.json
```


### Deploy the eks cluster 
Run the following command in terminal
```bash
eksctl create cluster -f 01-cluster-config.yaml
```

### Emable OIDC
Enable OIDC on your cluster so that IRSA can be used in the cluster. This allows associating kubernetes service account roles to aws iam roles to allow pods and services to access aws services. 

Set cluster name
```bash
export cluster_name=my-eks-cluster
```

Retrieve oidc_id
``` bash
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```

Check whether oidc has already been set up
```bash
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

If oidc has not been set up, then set it up
```bash
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```
