# Setup
- Secret data: ENV=DEV or PROD, SOURCE=YAML, GCP or VAULT
- Pod which reads the secret and provides it via the environment

# Case 1
## Objective
- Secret managed by team directly
- Secret provided as Sealed Secret

## Usage
### Seal k8s Secret
- Install sealed secrets controller on cluster via helm
```
helm repo add bitnami https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets-controller bitnami/sealed-secrets
```
- Install kubeseal, fetch pubkey from cluster
```
brew install kubeseal
kubeseal --fetch-cert --controller-name=sealed-secrets --controller-namespace=default > pub-cert.pem
```
- Seal Secret with kubeseal
```
kubeseal --cert=pub-cert.pem --format=yaml < prod-k8s-secret.yaml > sealed-secret.yaml
```
### Deploy sealed secret and app
```
kubectl apply -f sealed-secret.yaml
kubectl apply -nsealed-secret-app -f ../app/prod-reading-secret.yaml
```

# Case 2
## Objective
- Secret managed by Vault
- Secret provided by External Secret Operator

## Usage
- Install Consul
MAKE SURE YOU DO NOT HAVE A DEFAULT PV ON YOUR CLUSTER ALREADY
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install --values consul-values.yaml consul hashicorp/consul -n vault --create-namespace
```
- Install Vault
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install --values vault-values.yaml vault hashicorp/vault -n vault

export VAULT_ADDR=http://127.0.0.1:8200
kubectl port-forward svc/vault -n vault 8200:8200
```

- Setup Vault for the first time
```
vault operator init
vault operator unseal <key 1>
vault operator unseal <key 2>
vault operator unseal <key 3>
vault login <root token>
```

- Install ESO
```
helm repo add eso https://charts.external-secrets.io
helm install external-secrets eso/external-secrets --namespace eso --create-namespace
``` 

- Configure ESO to authenticate with Vault
```
vault auth enable kubernetes

k8s_host="$(kubectl exec vault-0 -n vault -- printenv | grep KUBERNETES_PORT_443_TCP_ADDR | cut -f 2- -d "=" | tr -d " ")"
k8s_port="443"            
k8s_cacert="$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 --decode)"
secret_name="$(kubectl get serviceaccount vault -n vault -o jsonpath='{.secrets[0].name}')"
tr_account_token="$(kubectl get secret ${secret_name} -n vault -o jsonpath='{.data.token}' | base64 --decode)"

vault write auth/kubernetes/config token_reviewer_jwt="${tr_account_token}" kubernetes_host="https://${k8s_host}:${k8s_port}" kubernetes_ca_cert="${k8s_cacert}" 
disable_issuer_verification=true
```
```
sa_secret_name="$(kubectl get serviceaccount external-secrets -n eso -o jsonpath='{.secrets[0].name}')"
sa_account_token="$(kubectl get secret ${sa_secret_name} -n eso -o jsonpath='{.data.token}' | base64 --decode)"

vault policy write eso-policy -<<EOF     
path "kv/data/test-secret"                                                  
{  capabilities = ["read"]                
}                         
EOF

vault write auth/kubernetes/role/eso-role \
    bound_service_account_names=external-secrets \
    bound_service_account_namespaces=eso \
    policies=eso-policy \
    ttl=24h

vault write auth/kubernetes/login role=eso-role jwt=$sa_account_token iss=https://kubernetes.default.svc.cluster.local
```
```
kubectl apply -f vault/secret-store.yaml
```

- Create secret in Vault
```
vault secrets enable -version=2 kv
vault kv put kv/test-secret ENV=PROD SOURCE=VAULT
```
- Create External Secret
```
kubectl apply -f vault/external-secret.yaml
```
