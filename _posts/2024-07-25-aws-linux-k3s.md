---
title: Install k3s on AWS Linux 2 for standalone k8s deployments
categories: [k8s]
tags: [eks,k8s,aws]
---
K3s is an option for deploying k8s when a full k8s environment may not be required, for instance for development and testing.  This post describe how to set up k3s on Amazon Linux 2 running on EC2. 

The following components are deployed:
- docker
- k3s
- ingress nginx
- helm
- argocd


Install docker, since this deployment will use docker as a container runtime. The default runtime is containerd, which is installed by default. This step can be skipped if using containerd runtime.
```bash
sudo yum update

sudo yum install docker

sudo usermod -a -G docker ec2-user

id ec2-user

newgrp docker

sudo systemctl enable docker.service
```


Set kubeconfig permissions to be accessible by non-root users
```bash
export K3S_KUBECONFIG_MODE="644"
```

Install k3s with docker and without Traefik, ingress-nginx will be used instead
```bash
curl -sfL https://get.k3s.io | sh -s - --docker --disable traefik
```


Symlink kubeconfig to default location
```bash
ln -s /etc/rancher/k3s/k3s.yaml ~/.kube/config
```


Install helm 
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Deploy Nginx Ingress.
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update

helm install internal-ingress ingress-nginx/ingress-nginx \
--namespace internal-ingress   \
--create-namespace             \
--set controller.ingressClassResource.name="nginx \
--set controller.ingressClass="nginx" \
--set controller.defaultBackendService.enabled=true   
```

Set up argo-values.yaml 
```yaml
cat <<EOF  > argo-values.yaml
global:
  # no domain specified so *:80 is routed to ingress on the host
  domain: 
configs:
  params:
    # allow non tls
    server.insecure: true

  secret:
    ## Argo expects the password in the secret to be bcrypt hashed. You can create this hash with
    ##  htpasswd -nbBC 10 "" 'password123ABC' | tr -d ':\n' | sed 's/$2y/$2a/'
    argocdServerAdminPassword: "$2a$10$3QBVJfaeB..rs4lwjkhwXuC1Uj.yj/hCWX3PnLxklnqyQERpOaCci"
    
server:
  ingress:
    enabled: true
    ingressClassName: nginx
    # allow non tls
    tls: false 
EOF
```

Add argo helm repository
```bash 
helm repo add argo https://argoproj.github.io/argo-helm
```

Update helm repos
```bash
helm repo update
```

Install Argo CD
```bash
helm install argocd argo/argo-cd -n argocd --create-namespace -f argo-values.yaml
```

Once deployed argo should be available on [http://localhost:80](http://localhost:80)