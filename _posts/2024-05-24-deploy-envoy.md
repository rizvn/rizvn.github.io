---
title: Deploy Envoy Gateway TCP Routes
categories: [k8s]
tags: [eks,k8s,aws, minikube, envoy]
---
TCP Routes in envoy gateway allow routing TCP traffic in managed way similar to http. 

This post is inspired by [this post](https://gateway.envoyproxy.io/v0.6.0/user/tcp-routing/#:~:text=TCPRoute%20provides%20a%20way%20to,to%20the%20Gateway%20API%20documentation) on envoy gateway site. 

This blog post will set up the following:

![alt text](/assets/images/envoy-gateway.png "envoy-diagram")

There are: 
- 2 services foo and bar they are in thier own namespaces
- 2 tcp routes within the namespaces
- gateway-01 class and assoociated proxy, the proxy is in the envoy-gatway-system namespace
- gateway definition with listeners in gateway-01 namespace.

First install envoy proxy system using helm
```bash
helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm --version v0.6.0 -n envoy-gateway-system --create-namespace
```

Create an envoy proxy with custom config and associated gateway class. The aws-loadbalancer annotations are optional
```bash 
kubectl create ns gateway-01
```

Create a gateway class
``` yaml 
kubectl apply -n gateway-01 -f - <<EOF 
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        replicas: 3
      envoyService:
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: instance
          service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
          service.beta.kubernetes.io/aws-load-balancer-internal: 'true'
          service.beta.kubernetes.io/aws-load-balancer-scheme: internal
          service.beta.kubernetes.io/aws-load-balancer-type: nlb
---
kind: GatewayClass
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: gateway-01
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: custom-proxy-config
    namespace: envoy-gateway-system
EOF
```


Define listeners for the gateway class. This tells the proxy to listen to these ports
```yaml
kubectl apply -n gateway-01 -f - <<EOF 
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-01
spec:
  # attach the gateway to the gateway class
  gatewayClassName: gateway-01
  listeners:
    # listens on 8088
    - name: foo-port
      protocol: TCP
      port: 8088

      # allow TCPRoute from any namespace to be attached to this listener
      allowedRoutes:
        namespaces:
          from: All
        kinds:
          - kind: TCPRoute

    # listens on 8089
    - name: bar-port
      protocol: TCP
      port: 8089

      # allow TCPRoute from any namespace to be attached to this listener
      allowedRoutes:
        namespaces:
          from: All
        kinds:
          - kind: TCPRoute
EOF
```

Create a foo service with deployment. The service echo http headers and service name of foo
Create the namespace
```bash 
kubectl create ns foo-ns
```

Create service and deployment
```yaml
kubectl apply -n foo-ns -f - <<EOF 
apiVersion: v1
kind: Service
metadata:
  name: foo-service
  labels:
    app: foo
spec:
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  selector:
    app: foo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: foo-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: foo
      version: v1
  template:
    metadata:
      labels:
        app: foo
        version: v1
    spec:
      containers:
        - image: gcr.io/k8s-staging-ingressconformance/echoserver:v20221109-7ee2f3e
          imagePullPolicy: IfNotPresent
          name: foo
          ports:
            - containerPort: 3000
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: SERVICE_NAME
              value: foo
EOF
```

Create a bar service with deployment. The service echo http headers and service name of bar

Create namespace
```bash 
kubectl create ns bar-ns
```

Create service and deployment
```yaml
kubectl apply -n bar-ns -f - <<EOF 
apiVersion: v1
kind: Service
metadata:
  name: bar-service
  labels:
    app: bar
spec:
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  selector:
    app: bar
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bar-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bar
      version: v1
  template:
    metadata:
      labels:
        app: bar
        version: v1
    spec:
      containers:
        - image: gcr.io/k8s-staging-ingressconformance/echoserver:v20221109-7ee2f3e
          imagePullPolicy: IfNotPresent
          name: bar
          ports:
            - containerPort: 3000
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: SERVICE_NAME
              value: bar
EOF
```


Create tcp route for foo
```yaml
kubectl apply -n foo-ns -f - <<EOF 
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: foo-tcp-route
spec:
  parentRefs:
    # the name of the Gateway and listener
    - name: gateway-01
      sectionName: foo-port
      namespace: gateway-01
  rules:
    - backendRefs:
        # destination service & port
        - name: foo-service
          port: 3000
EOF
```

Create tcp route for bar
```yaml
kubectl apply -n bar-ns -f - <<EOF 
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: bar-tcp-route
spec:
  parentRefs:
    # the name of the Gateway and listener
    - name: gateway-01
      sectionName: bar-port
      namespace: gateway-01
  rules:
    - backendRefs:
        # destination service & port
        - name: bar-service
          port: 3000
EOF
```


If using minikube for testing, use minikube tunnel to associate ip to envoy proxy serioce
```bash
minikube tunnel
```

Test
```bash 
# to call foo service
curl http://10.98.159.232:8088
```
output example:
```json
{
 "path": "/",
 "host": "10.98.159.232:8088",
 "method": "GET",
 "proto": "HTTP/1.1",
 "headers": {
  "Accept": [
   "*/*"
  ],
  "User-Agent": [
   "curl/8.5.0"
  ]
 },
 "namespace": "foo-ns",
 "ingress": "",
 "service": "foo",
 "pod": "foo-deployment-5c85498794-fxfrm"
}
```

Test bar service
```bash
curl http://10.98.159.232:8088
```

output example
```json
{
 "path": "/",
 "host": "10.98.159.232:8089",
 "method": "GET",
 "proto": "HTTP/1.1",
 "headers": {
  "Accept": [
   "*/*"
  ],
  "User-Agent": [
   "curl/8.5.0"
  ]
 },
 "namespace": "bar-ns",
 "ingress": "",
 "service": "bar",
 "pod": "bar-deployment-5fdb677b4f-mnhhf"
}
```