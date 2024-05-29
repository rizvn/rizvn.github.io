---
title: Deploy Docker Registry Mirror on AWS EKS 
categories: [k8s]
tags: [eks,k8s,aws, minikube, docker, registry]
---
Below is how docker registry can be deployed with AWS EKS. The default AWS EKS images use containerd rather than moby as the container engine.

This blog post was inspired by [this post](https://docs.docker.com/docker-hub/mirror/) on the docker website.

Create namespace for docker registry
```bash
kubectl create ns docker-registry-mirror
```

Deploy registry mirror. 
**Optionally**: comment out username and password in confiigmap to authenticate with dockerhub.
```yaml
kubectl apply -n docker-registry-mirror -f - <<EOF 
apiVersion: v1
kind: Service
metadata:
  name: docker-registry
spec:
  selector:
    app: docker-registry
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: docker-registry
spec:
  serviceName: docker-registry
  replicas: 1
  selector:
    matchLabels:
      app: docker-registry
  template:
    metadata:
      labels:
        app: docker-registry
    spec:
      containers:
        - name: docker-registry
          # alternative ecr image: public.ecr.aws/docker/library/registry:2
          image: registry:2
          ports:
            - containerPort: 5000
          volumeMounts:
            - name: registry-storage
              mountPath: /var/lib/registry
            - name: registry-config
              mountPath: /etc/docker/registry/config.yml
              subPath: config.yml
      volumes:
        - name: registry-config
          configMap:
            name: docker-registry-config
  volumeClaimTemplates:
    - metadata:
        name: registry-storage
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 300Gi
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: docker-registry-config
data:
  config.yml: |
    version: 0.1
    log:
      fields:
        service: registry
    storage:
      cache:
        blobdescriptor: inmemory
      filesystem:
        rootdirectory: /var/lib/registry
    proxy:
      remoteurl: https://registry-1.docker.io
      # username: ""
      # password: ""
    http:
      addr: :5000
      headers:
        X-Content-Type-Options: [nosniff]
EOF
```

Get cluster ip of service docker-registry in namespace docker-registry-mirror
```bash
export DOCKER_REGISTRY_IP=$(kubectl get svc docker-registry -n docker-registry-mirror -o jsonpath='{.spec.clusterIP}')
```

Create a Daemonset to configure containerd to use the docker registry mirror. This will run on each node and update the containerd config to use the pull through mirror.
```yaml
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    name: containerd-dockerhub-registry-config
  name: containerd-dockerhub-registry-config
spec:
  selector:
    matchLabels:
      name: containerd-dockerhub-registry-config
  template:
    metadata:
      labels:
        name: containerd-dockerhub-registry-config
    spec:
      containers:
        # alternative ecr image: public.ecr.aws/docker/library/alpine:latest
        - image: alpine:latest
          name: containerd-dockerhub-registry-config
          command:
            - /bin/sh
            - -c
            - |
              #!/bin/sh
              set -uo pipefail
              while true; do
                cat << EOF > /etc/containerd/certs.d/docker.io/hosts.toml
              server = "https://registry-1.docker.io"

              [host."http://${DOCKER_REGISTRY_IP}:5000"]
                capabilities = ["pull", "resolve"]
              EOF
                sleep 3600s
              done
          volumeMounts:
            - mountPath: /etc/containerd/certs.d/docker.io
              name: docker-io-dir
      volumes:
        - name: docker-io-dir
          hostPath:
            path: /etc/containerd/certs.d/docker.io/
EOF
```


Now all nodes will pull images through mirror, allowing images to be cached so a round trip to dockerhub is not always required. 

