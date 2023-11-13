---
title: Hello 
categories: [blog]
tags: [eks, kubernetes, k8s]
---

# Setting up an eks cluster in aws

```bash
cat > cluster-config.yaml << EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: test-cluster
  region: eu-west-1

vpc:
  subnets:
    private:
      eu-west-1a: { id: subnet-0ff156e0c4a6d300c }
      eu-west-1b: { id: subnet-0549cdab573695c03 }
      eu-west-1c: { id: subnet-0426fb4a607393184 }


nodeGroups:
- name: ng-1-workers
  labels: { role: workers }
  instanceType: m5.xlarge
  desiredCapacity: 10
  privateNetworking: true
- name: ng-2-builders
  labels: { role: builders }
  instanceType: m5.2xlarge
  desiredCapacity: 2
  privateNetworking: true
  iam:
    withAddonPolicies:
      imageBuilder: true
EOF
```

Create the cluster 
```bash
#create the cluster
eksctl create cluster 
--name some-name \ 
-f cluster-config.yaml
```

