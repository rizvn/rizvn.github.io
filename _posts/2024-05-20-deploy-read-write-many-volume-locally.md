---
title: Deploy ReadWriteMany volume on local K8s for testing
categories: [k8s]
tags: [eks,k8s,aws,nfs,samba,EFS]
---
There are several use cases where you may need to share folders between applications. In the cloud hosted kubernetes this requirement can be achieved with cloud native shared disks such as AWS EFS. However we may want to deploy something locally to help test containers. 

This solution creates a mock nfs storage class using the smb protocol, in order to dynamically provision ReadWriteMany volumes
on local kubernetes clusters, such as rancher and minikube, for local testing and development.


create namespace
``` bash
kubectl create ns mock-nfs
```

Create credentials
``` yaml
kubectl apply -n mock-nfs -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: smbcreds
type: Opaque
stringData:
  username: testuser
  password: testpass
EOF
```

Create Server 
```yaml
kubectl apply -n mock-nfs -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: samba-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: samba
  template:
    metadata:
      labels:
        app: samba
    spec:
      containers:
        - name: samba
          image: dperson/samba:latest
          env:
            - name: USERID
              value: "0"
            - name: GROUPID
              value: "0"
            - name: TZ
              value: "Europe/London"
            - name: USER
              value: "testuser;testpass"
            - name: SHARE
              value:  "share;/share;yes;no;no;testuser;;;"
          ports:
            - containerPort: 445
          volumeMounts:
            - name: share-pvc
              mountPath: /share
      volumes:
        - name: share-pvc
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: samba
spec:
  selector:
    app: samba
  ports:
    - protocol: TCP
      port: 445
      targetPort: 445
      name: samba-tcp
  type: ClusterIP
EOF
```

Deploy provisioner on kubernetes
``` bash
helm repo add csi-driver-smb https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/charts

helm install csi-driver-smb csi-driver-smb/csi-driver-smb  --version v1.11.0 -n mock-nfs
```


Create the storage class
```yaml
kubectl apply -n mock-nfs -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs
provisioner: smb.csi.k8s.io
parameters:
  # name-inside-cluser/share
  source: "//samba.mock-nfs.svc.cluster.local/share"

  # if csi.storage.k8s.io/provisioner-secret is provided, will create a sub directory
  # with PV name under source
  csi.storage.k8s.io/provisioner-secret-name: "smbcreds"
  csi.storage.k8s.io/provisioner-secret-namespace: "mock-nfs"
  csi.storage.k8s.io/node-stage-secret-name: "smbcreds"
  csi.storage.k8s.io/node-stage-secret-namespace: "mock-nfs"
reclaimPolicy: Delete  # available values: Delete, Retain
volumeBindingMode: Immediate
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=0
  - gid=0
EOF
```

To test this you can create a shared volume like this. The pvc does not need to be in the same namespace, it will be be created by the csi-smb driver 
```yaml
kubectl apply -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: shared-vol
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
EOF
```


Then mount it 2 pods.

Pod 1
Mounts the shared-pvc to `/opt`
this pod sleeps write `hello` to `/opt/hello`
```yaml
kubectl apply -f - <<EOF
kind: Pod
apiVersion: v1
metadata:
  name: test-pod1
spec:
  containers:
    - name: test-pod1
      image: busybox:stable
      resources:
        requests:
          cpu: 100m
          memory: 50Mi
        limits:
          cpu: 100m
          memory: 50Mi
      command:
        - "/bin/sh"
      args:
        - "-c"
        - "echo hello > /opt/hello && sleep 3600"
      volumeMounts:
        - name: shared-pvc
          mountPath: "/opt"
  restartPolicy: "Never"
  volumes:
    - name: shared-pvc
      persistentVolumeClaim:
        claimName: shared-vol
EOF
```

Pod 2 
Mounts the shared-pvc to `/opt`
this pod sleeps for 10 seconds prints contents of `/opt/hello`
```yaml
kubectl apply -f - <<EOF
kind: Pod
apiVersion: v1
metadata:
  name: test-pod2
spec:
  containers:
    - name: test-pod2
      image: busybox:stable
      resources:
        requests:
          cpu: 100m
          memory: 50Mi
        limits:
          cpu: 100m
          memory: 50Mi
      command:
        - "/bin/sh"
      args:
        - "-c"
        - " sleep 10 && echo $(cat /opt/hello) && sleep 3600"
      volumeMounts:
        - name: shared-pvc
          mountPath: "/opt"
  restartPolicy: "Never"
  volumes:
    - name: shared-pvc
      persistentVolumeClaim:
        claimName: shared-vol
EOF
```
