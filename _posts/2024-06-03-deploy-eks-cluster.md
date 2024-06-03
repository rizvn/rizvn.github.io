---
title: Create an EKS cluster with eksctl and aws cli
categories: [k8s]
tags: [eks,k8s,aws,eksctl]
---
### Install aws cli 
Install  aws cli from [here](https://aws.amazon.com/cli/)

### Install eksctl
Install eksctl from  [here](https://eksctl.io/)


### Set your aws account number 
```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```



### Define worker policy for aws worker nodes
Create policy for worker nodes in the cluster to able to pull images from your private ecr repository.

```json
cat << EOF > worker-iam-policy.json
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
EOF
```

### Deploy AWS policy for worker nodes
```bash
aws iam create-policy --policy-name worker-policy --policy-document worker-iam-policy.json
```

### Define cluster config
```yaml
cat << EOF > 01-cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-eks-cluster
  region: eu-west-2

# to enable oidc (irsa)
iam:
  withOIDC: true


  # service accounts and policy from common operators
  serviceAccounts:
  - metadata:
      name: aws-load-balancer-controller
      namespace: kube-system
    wellKnownPolicies:
      awsLoadBalancerController: true
  - metadata:
      name: ebs-csi-controller-sa
      namespace: kube-system
    wellKnownPolicies:
      ebsCSIController: true
  - metadata:
      name: efs-csi-controller-sa
      namespace: kube-system
    wellKnownPolicies:
      efsCSIController: true
  - metadata:
      name: external-dns
      namespace: kube-system
    wellKnownPolicies:
      externalDNS: true
  - metadata:
      name: cert-manager
      namespace: cert-manager
    wellKnownPolicies:
      certManager: true
  - metadata:
      name: cluster-autoscaler
      namespace: kube-system
      labels: {aws-usage: "cluster-ops"}
    wellKnownPolicies:
      autoScaler: true
  - metadata:
      name: autoscaler-service
      namespace: kube-system
    attachPolicy: # inline policy can be defined along with `attachPolicyARNs`
      Version: "2012-10-17"
      Statement:
      - Effect: Allow
        Action:
        - "autoscaling:DescribeAutoScalingGroups"
        - "autoscaling:DescribeAutoScalingInstances"
        - "autoscaling:DescribeLaunchConfigurations"
        - "autoscaling:DescribeTags"
        - "autoscaling:SetDesiredCapacity"
        - "autoscaling:TerminateInstanceInAutoScalingGroup"
        - "ec2:DescribeLaunchTemplateVersions"
        Resource: '*'


# to enable network policies
addons:
  - name: vpc-cni
    version: latest
    attachPolicyARNs:
      - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

# deploy karpenter
karpenter:
  version: 'v0.20.0' # Exact version must be provided
  createServiceAccount: true # default is false
  withSpotInterruptionQueue: true # adds all required policies and rules for supporting Spot Interruption Queue, default is false


managedNodeGroups:
  - name: workers
    labels: { role: workers }
    desiredCapacity: 2
    volumeSize: 100
    instancesDistribution:
      maxPrice: 0.085 # Set an appropriate max price for Spot instances
    availabilityZones: ["eu-west-2a", "eu-west-2b", "eu-west-2c"]
    privateNetworking: true # If you want to use private networking for the nodes
    iam:
      attachPolicyARNs:
        - arn:aws:iam:${AWS_ACCOUNT_ID}:aws:policy/worker-policy

    # instance types for the nodes
    instanceType:
      - m5.large
      - c5.large


    # Specify subnets for each Availability Zone
    subnets:
      - subnet-0-az-a
      - subnet-1-az-b
      - subnet-2-az-c
    
    # taint nodes e.g only allow certain type of node group
    #taints:
    #  - key: ingress-only
    #    value: "true"
    #    effect: NoSchedule
    #  - key: ingress-only
    #    value: "true"
    #    effect: NoExecute

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

    nat:
      gateway: HighlyAvailable 
EOF
```




### Deploy the eks cluster 
```bash
eksctl create cluster -f 01-cluster-config.yaml
```