---
title: Create an EKS cluster with eksctl and aws cli
categories: [k8s]
tags: [eks,k8s,aws,eksctl]
---
### Install aws cli 
Install  aws cli from [here](https://aws.amazon.com/cli/)

### Install eksctl
Install eksctl from  [here](https://eksctl.io/)

### Define the cluster config
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
        - arn:aws:iam:YOUR-ACCOUNT-NUMBER:aws:policy/worker-policy

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

**Replace** YOUR-ACCOUNT-NUMBER with your aws account number in the file above this can be done using the following **sed** command

```bash
sed -i 's/arn:aws:iam:YOUR-ACCOUNT-NUMBER:/arn:aws:iam:111111111:/g' 01-cluster-config.yaml
```


### Define worker policy for aws worker nodes
Create policy for worker nodes in the cluster to able to pull images from your private ecr repository.

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

### Deploy AWS policy for worker nodes
Run the following command in terminal
```bash
aws iam create-policy --policy-name worker-policy --policy-document 02-worker-iam-policy.json
```


### Deploy the eks cluster 
Run the following command in terminal
```bash
eksctl create cluster -f 01-cluster-config.yaml
```

### Enable OIDC for EKS cluster
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
