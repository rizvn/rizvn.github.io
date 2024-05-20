---
title: Integrate Minikube with AWS ECR
categories: [k8s]
tags: [eks,k8s,aws,minkikube]
---
When testing images locally you may need to pull images from aws in your deployments. AWS ECR is a dokcer  

Create a namespace in which the pods will be deployed
```bash
kubetl create ns test
```

Remove secret named regcred in test namespace, then create it. 

**Note**
This should be repeated every 6 hours to ensure that the ECR token does not expire. However this can be run through a cron job or prior to pulling any changed images where 6 hours have elapsed. 
```bash
# change to your account id
AWS_ACCOUNT=123456789

# change to your eqgion
AWS_REGION=eu-west-1

# change to your namespace
NAMESPACE=test

kubectl delete secret --ignore-not-found regcred -n ${NAMESPACE} 

kubectl create secret -n ${NAMESPACE}  docker-registry regcred \
    --docker-server=${AWS_ACCOUNT}.dkr.ecr.<region>.amazonaws.com \
    --docker-username=AWS \
    --docker-password=$(aws ecr get-login-password --region ${AWS_REGION})
```


Use Imagepull secret to get the image 
```yaml
spec:
  container:
    - name: my-container
      image: some-address.com/my-container 
imagePullSecrets:
  - name: regcred
```
