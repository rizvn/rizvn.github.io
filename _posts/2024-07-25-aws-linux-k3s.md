---
title: Install k3s on AWS Linux 2 for standalone k8s deployments
categories: [k8s]
tags: [eks,k8s,aws]
---
K3s is an option for deploying k8s when a full k8s environment may not be required, for instance for development and testing.  This post describe how to set up k3s on Amazon Linux 2 running on EC2. 

The following components are deployed:
- k3s
- ingress nginx
- helm
- argocd



Set kubeconfig permissions to be accessible by non-root users
```bash
export K3S_KUBECONFIG_MODE="644"
```

Install k3s with docker and without Traefik, ingress-nginx will be used instead
```bash
curl -sfL https://get.k3s.io | sh -s - --disable traefik
```


Symlink kubeconfig to default location
```bash
mkdir -p ~/.kube
ln -s /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

Check k3s status
```bash
kubectl get nodes
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
--set controller.ingressClassResource.name="nginx" \
--set controller.ingressClass="nginx" \
--set controller.defaultBackendService.enabled=true
```

Set up argo-values.yaml.
Remember to escape the $ with a backslash (\$), since cat is used to create this file
```yaml
cat <<EOF  > argo-values.yaml
global:
  # no domain specified so *:80 is routed to argo through the ingress
  domain: 
configs:
  params:
    # allow non tls
    server.insecure: true

  secret:
    ## Argo expects the password in the secret to be bcrypt hashed. You can create this hash with
    ##  htpasswd -nbBC 10 "" 'password123ABC' | tr -d ':\n' | sed 's/$2y/$2a/'
    argocdServerAdminPassword: "\$2a\$10\$vvKImPs2Bc..vjaeM3sHW.aVADxpLmQmO8XFH/8ZbL0pWonBYwwv2"
    
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
credentials are `admin` and `password123ABC`
