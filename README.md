# Setup
- Secret data: FOO=BAR, SOURCE=YAML, GCP or VAULT
- Pod which reads the secret and provides it via the environment

# Case 1
## Objective
- Secret managed by team directly
- Secret provided as Sealed Secret

## Usage
### Seal Secret for remote cluster usage
- Install sealed secrets controller on remote cluster via helm
```
helm repo add bitnami https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets bitnami/sealed-secrets
```
- Install kubeseal, fetch pubkey from cluster
```
brew install kubeseal
kubeseal --fetch-cert --controller-name=sealed-secrets --controller-namespace=default > remote-cluster/pub-cert.pem
```
- Seal Secret with kubeseal
```
kubeseal --cert=remote-cluster/pub-cert.pem --format=yaml < app/k8s-secret.yaml > app/sealed-secret.yaml
```
### make sealed secret usable on local cluster
- Fetch private keys from remote cluster
```
kubectl get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > remote-cluster/master.key
```
- Install sealed secrets on local cluster (done via tilt)
- Update private keys on local cluster's sealed secrets controller
```
kubectl apply -f remote-cluster/master.key
kubectl delete pod -l name=sealed-secrets
```

# Case 2
## Objective
- Secret managed by Cloud Provider (GCP)
- Secret provided by External Secret Operator

- Create GCP Service Account with necessary role
```
gcloud iam service-accounts create external-secrets \
    --project=GCP_PROJECT_ID
gcloud projects add-iam-policy-binding GCP_PROJECT_ID \
    --member "serviceAccount:external-secrets@GCP_PROJECT_ID.iam.gserviceaccount.com" \
    --role "roles/secretmanager.secretAccessor"
```
- Create a cluster with Workload Identity enabled
```
gcloud container clusters create CLUSTER_NAME \
    --region=COMPUTE_REGION \
    --workload-pool=secret-management-talk.svc.id.goog
```

- Install ESO
```
helm repo add eso https://charts.external-secrets.io
helm install external-secrets eso/external-secrets --namespace eso --create-namespace
```
- Allow K8s SA to impersonate GCP SA
```
gcloud iam service-accounts add-iam-policy-binding external-secrets@GCP_PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:GCP_PROJECT_ID.svc.id.goog[eso/external-secrets]"
```
- Add GCP SA json key to cluster
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: gcpsm-secret
  labels:
    type: gcpsm
type: Opaque
stringData:
  secret-access-credentials: |-
    // This is where the jsonkey goes
EOF
```
- Allow ESO to connect to GCP & retrieve the secret from Google Secret Manager
```
kubectl apply -f eso/secret-store.yaml
kubectl apply -f eso/gcp-secret.yaml
```

- Create GCP Secrets
```
printf "BAR" | gcloud secrets create FOO --data-file=-
printf "GCP" | gcloud secrets create SOURCE --data-file=-
```

- Create External Secret
```
kubectl apply -f eso/external-secret.yaml
```

# Case 3
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
helm install  --values vault-values.yaml vault hashicorp/vault -n vault

export VAULT_ADDR=http://127.0.0.1:8200
kubectl port-forward svc/vault -n vault 8200:8200 &
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
vault policy write eso-policy -<<EOF     
path "kv/data/test-secret"                                                  
{  capabilities = ["read"]                
}                         
EOF
```
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
demo_secret_name="$(kubectl get serviceaccount external-secrets -n eso -o jsonpath='{.secrets[0].name}')"
demo_account_token="$(kubectl get secret ${demo_secret_name} -n eso -o jsonpath='{.data.token}' | base64 --decode)"                  

vault write auth/kubernetes/role/eso-role \
    bound_service_account_names=external-secrets \
    bound_service_account_namespaces=es \
    policies=eso-policy \
    ttl=24h

vault write auth/kubernetes/login role=eso-role jwt=$demo_account_token iss=https://kubernetes.default.svc.cluster.local
```

- Create secret in Vault
```
vault secrets enable -version=2 kv
vault kv put kv/test-secret FOO=BAR SOURCE=VAULT
```
- Create External Secret
```
kubectl apply -f vault/external-secret.yaml
```
