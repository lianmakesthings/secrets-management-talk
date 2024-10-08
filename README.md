# Setup
- Secret data: ENV=DEV or PROD, SOURCE=YAML, GCP or VAULT
- Pod which reads the secret and provides it via the environment

# Case 1
## Objective
- Secret managed by team directly
- Secret provided as Sealed Secret

## Usage
Navigate to the subfolder
```
cd sealed-secrets
```
### With vcluster & DevSpace (on dev)
```
vcluster create sealed-secrets -nsealed-secrets
devspace dev
```

### Manually (on prod)
#### Seal k8s Secret
- Install sealed secrets controller on cluster via helm
```
helm repo add bitnami https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets-controller bitnami/sealed-secrets
```
- Install kubeseal, fetch pubkey from cluster
```
brew install kubeseal
kubeseal --fetch-cert --controller-namespace=default > pub-cert.pem
```
- Seal Secret with kubeseal
```
kubeseal --cert=pub-cert.pem --format=yaml < prod-k8s-secret.yaml > sealed-secret.yaml
```
#### Deploy sealed secret and app
```
kubectl create namespace sealed-secret-app
kubectl apply -f sealed-secret.yaml
kubectl apply -nsealed-secret-app -f ../app/pod-reading-secret.yaml
```

# Case 2
## Objective
- Secret managed by Vault
- Secret provided by External Secret Operator

## Usage
Navigate to the subfolder
```
cd vault
```
### With vcluster & DevSpace (on dev)
```
vcluster create vault-eso -nvault-eso
devspace dev
```

### Manually (on prod)
#### Install & Setup Vault
- Install Consul
MAKE SURE YOU DO NOT HAVE A DEFAULT PV ON YOUR CLUSTER ALREADY
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install --values consul-values.yaml consul hashicorp/consul -n vault --create-namespace
```
- Install Vault
```
helm install --values vault-prod-values.yaml vault hashicorp/vault -n vault
kubectl port-forward svc/vault -n vault 8200:8200
```

- Install Vault CLI
https://developer.hashicorp.com/vault/docs/install

- Setup Vault for the first time
```
export VAULT_ADDR=http://127.0.0.1:8200
vault operator init
```
The last command will return five keys and a root token that can be used to unseal the vault and interact with it as shown below
```
vault operator unseal <key 1>
vault operator unseal <key 2>
vault operator unseal <key 3>
vault login <root token>
```

#### Install ESO
```
helm repo add eso https://charts.external-secrets.io
helm install external-secrets-operator eso/external-secrets --namespace eso --create-namespace
``` 

#### Authorize ESO for Vault
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
vault policy write eso-policy -<<EOF     
path "kv/data/test-secret"                                                  
{  capabilities = ["read"]                
}                         
EOF

vault write auth/kubernetes/role/eso-role \
    bound_service_account_names=external-secrets-operator \
    bound_service_account_namespaces=eso \
    policies=eso-policy \
    ttl=24h

```
```
kubectl apply -f secret-store.yaml
```

#### Deploy Secret & App
- Create secret in Vault
```
vault secrets enable -version=2 kv
vault kv put kv/test-secret ENV=PROD SOURCE=VAULT
```
- Create External Secret
```
kubectl create namespace vault-eso-app
kubectl apply -f external-secret.yaml
```
- Read Secret via app
```
kubectl apply -nvault-eso-app -f ../app/pod-reading-secret.yaml
```