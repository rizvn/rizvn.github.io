---
title: TooManyTargets AWS NLB
categories: [k8s]
tags: [eks,k8s,aws, nlb, alb]
---
On AWS EKS, loadbalancer's are creatd by setting the the service type to LoadBalancer. When created each port defined on the service will be associated with a TargetGroup. Each TargetGroup will be associated with an Node (EC2 isnstance) within the cluster. AWS NLB has target limit of 500, so if you have 10 ports and 51 nodes in your eks cluster you will quickly pass the limit on an NLB. 

The service will report the following error event:
```yaml
TooManyTargets: You cannot have more than 500 targets per network load balancer per Availability Zone
```


A way around this is to not use every node as target within the cluster. This can be done by attaching labels to nodes which can act as targets. This can be done when creating the nodegroup for example:
```bash
aws eks create-nodegroup \
    --cluster-name my-cluster \
    --nodegroup-name ingress-only-ng \
    --labels allow-ingress=true \
    --node-role <node-instance-role> \
    --subnets <subnet-1> <subnet-2> ... 
```

AWS ALB controller provides an annotation that should be placed on the service.
```
service.beta.kubernetes.io/aws-load-balancer-target-node-labels: "allow-ingress=true"
```
[See here](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/service/annotations/#target-node-labels)

Kubectl example:
```bash
kubectl patch svc my-service -n my-ns -p '{"metadata":{"annotations":{"service.beta.kubernetes.io/aws-load-balancer-target-node-labels":"allow-ingress=true"}}}'
```

Now the only nodes with *allow-ingress=true* labels will be targets within the target groups.
