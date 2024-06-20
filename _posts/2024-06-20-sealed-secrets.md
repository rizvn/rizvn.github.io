---
title: Bitnami Sealed Secrets in K8s
categories: [k8s]
tags: [eks,k8s,aws]
---
Bitnami's sealed secrets allow secure storage of secrets in plain text such as in Git repos. The secrets are encrypted using a public key, the private key stays in the cluster, only the operator running in the cluster can decrypt. Sealed secrets also rotates the keypair's every 30 days. 

- The standard k8s secret is passed into `kubeseal`, which produces `SealedSecret` yaml 
- The SealedSecret yaml is applied to the cluster using `kubectl`
- The Bitnami operator watches for `SealedSecret` resources
- The Bitnami operator converts `SealedSecret` back to standard k8s `Secret`

Apps continue working with existing k8 `Secret`. 


### SetUP
Add helm repository for sealed-secrets
```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
```

Create a values file for the sealed-secrets helm chart, this will enable the ingress and set the
hostname and path for the cert. *This assumed there is an ingress class named nginx n the cluster. Change the ingressClassName or use local file approach outlined below if it  isn't.*
```yaml
cat <<EOF > sealed-secrets-values.yaml
fullnameOverride: sealed-secrets
ingress:
  enabled: true
  pathType: ImplementationSpecific
  ingressClassName: nginx
  hostname: sealed-secrets.k8s.local
  path: /v1/cert.pem
EOF
```

Install the sealed-secrets helm chart with the values file
```bash
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system \
--values sealed-secrets-values.yaml
```

Download and install kubeseal cli from
```bash
https://github.com/bitnami-labs/sealed-secrets/releases
```

Create a standard k8s secret resource
```yaml
cat <<EOF > secret.yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: null
  name: my-secret
stringData:
  username: foo
  password: bar
EOF
```  


Seal a k8s secret resource 
```bash
cat secret.yaml | kubeseal --cert http://sealed-secrets.k8s.local/v1/cert.pem \
--controller-namespace kube-system \
--controller-name sealed-secrets \
-o yaml > sealed-secret.yaml
```

--cert can be url or local file, use **SEALED_SECRETS_CERT** env var to omit --cert option, 
  the key will rotate every 30 days so it is recommended to use the url


Apply the sealed secret
```bash
kubectl apply -f sealed-secret.yaml
```


To update sealed secret, create a secret with only the fields that need to be updated
```yaml
cat <<EOF > secret-update.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
stringData:

  # new password
  password: newpassword
EOF
```

Merge the secret
```bash
cat secret-update.yaml | kubeseal --cert http://sealed-secrets.k8s.local/v1/cert.pem \
--controller-namespace kube-system \
--controller-name sealed-secrets \
--format yaml \
--merge-into sealed-secret.yaml


kubectl apply -f sealed-secret.yaml
```

To delete a key in the sealed secret just remove the value from the sealed-secret.yaml file and apply


### Use Local file instead of URL to encrypt secrets
get the public key (for offline usage not recommended, due to key rotation)
```bash
kubeseal  \
--controller-name=sealed-secrets \
--controller-namespace=kube-system       
--fetch-cert > cert.pem

```

Use cert.pem to seal 
```
cat secret.yaml | kubeseal --cert cert.pem \
--controller-namespace kube-system \
--controller-name sealed-secrets \
-o yaml > sealed-secret.yaml
```


### Backup sealed-secrets key
Backup the sealed-secrets key. `sealed-secrets-keyfkndl` is dynamically generated
```bash 
kubectl get secrets sealed-secrets-keyfkndl -n kube-system -o json | jq ".data | map_values(@base64d)"
```

This will produce the following:
``` json 
{
  "tls.crt": "-----BEGIN CERTIFICATE-----\nMIIE3DCCAsQCCQCgdNszn/dUUTANBgkqhkiG9w0BAQsFADAwMRYwFA...\n-----END CERTIFICATE-----\n",
  "tls.key": "-----BEGIN PRIVATE KEY-----\nMIIJQwIBADANBgkqhkiG9w0BAQEFAASCCS0wggkpAgEAAoICAQDAFYgUZStmW6Zo\n...\n-----END PRIVATE KEY-----\n"
}
```


### Re Encrypt
Re encrypt secrets, when cleaning up old keys, or when the key is compromised
```bash
kubeseal \
--controller-name=sealed-secrets \
--controller-namespace=kube-system -\
-re-encrypt -o yaml < sealed-secret.yaml  > new-sealed-secret.yaml
```

Apply new sealed secret
```bash
kubectl apply -f new-sealed-secret.yaml
```


