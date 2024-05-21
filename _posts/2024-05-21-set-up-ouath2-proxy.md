---
title: Set up oauth2 proxy with Azure AD
categories: [k8s]
tags: [eks,k8s,aws,oauth, azure, oidc]
---

#### Set up oauth2 proxy with Azure AD

Add Helm Repo to cluster
```bash
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
```

Generate cookie secret this is used to encrypt cookies, below is a bash command to randomly generate a string
```bash
dd if=/dev/urandom bs=32 count=1 2>/dev/null | base64 | tr -d -- '\n' | tr -- '+/' '-_' ; echo
```

Create values file for auth configuration `oauth2-proxy-values.yaml`
```yaml
cat << EOF > oauth2-proxy-values.yaml 
config:
  clientID: "xxx"
  clientSecret: "xxx"
  cookieSecret: "xxx"
  configFile: |-
    provider = "azure"
    azure_tenant="xxx"
    provider_display_name = "azure"  
    upstreams = [ "file:///dev/null" ]
    cookie_domains = ["test.k8s.local"]
    email_domains= ["*"]
    whitelist_domains = ["test.k8s.local"]
    http_address = "0.0.0.0.4180"
    pass_authorization_header="false"
    set_xauthrequest = "false"
    cookie_secure = "false"
    oidc_issuer_url = "https://login.microsoftonline.com/xxxxx-xxxx-xxx/v2.0"
    redirect_url="http://test.k8s.local/oauth2/callback"
    skip_jwt_bearer_tokens = "false""
sessionStorage:
  type: redis
  redis:
    passwordKey: "redis-password"
    clientType: "standalone"
redis:
  enabled: true
EOF
```
Replace the following:
1. clientID -provided by customer from their idp
2. clientScret - provided by customer from their idp
3. cookieSecret - base64 encoded random string generate above
4. azure_tenant - provided by customer from their idp
5. cookie_domains - domain of the where the oauth proxy and app
6. whitelist_domains - domain of the where the oauth proxy and app
7. oidc_issuer_url -  provided by customer from their idp
8. redirect_url - callback url to oauth proxy will be in the format http://<domain>/oauth2/callback, in this case it will be http://test.k8s.local/oauth2/callback

Deploy helm chart with values 
```bash 
helm install oauth-proxy oauth2-proxy/oauth2-proxy -n outh-proxy --values oauth2-proxy-values.yaml 
```



Create ingress for oauth proxy. Minikube can be used to test this set up, when nginx can be exposed using `minikube tunnel`.
```yaml
kubectl appky -n outh-proxy -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: oauth-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: test.k8s.local
      http:
        paths:
          - path: /oauth2
            pathType: Prefix
            backend:
              service:
                name: oauth-proxy-oauth2-proxy
                port:
                  number: 80
EOF
```


Now the follwing annotations need added to any ingress to use this proxy
```yaml
annotations:
  nginx.ingress.kubernetes.io/auth-response-headers: 'X-Auth-Request-User,X-Auth-Request-Email'
  nginx.ingress.kubernetes.io/auth-signin: http://test.k8s.local/oauth2/start?rd=https://$host$request_uri
  nginx.ingress.kubernetes.io/auth-url: 'http://test.k8s.local/oauth2/auth'
```