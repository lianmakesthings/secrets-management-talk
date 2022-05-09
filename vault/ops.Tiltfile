load('ext://helm_resource', 'helm_resource', 'helm_repo')

# Installing Hashicorp Vault
helm_repo('hashicorp', 'https://helm.releases.hashicorp.com', labels=['vault'])
helm_resource('consul', 'hashicorp/consul', namespace='vault', flags=['--create-namespace', '--values', 'consul-values.yaml'], labels=['vault'])
helm_resource('vault', 'hashicorp/vault', namespace='vault', resource_deps=['consul'], flags=['--create-namespace', '--values', 'vault-values.yaml'], labels=['vault'])
os.putenv('VAULT_ADDR', 'http://127.0.0.1:8200')
local_resource(
  name='forward-vault',
  serve_cmd='kubectl port-forward svc/vault -n vault 8200:8200',
  resource_deps=['vault'],
  labels=['vault']
)

# Installing External Secrets Operator
helm_repo('eso', 'https://charts.external-secrets.io', labels=['eso'])
helm_resource('external-secrets-operator', 'eso/external-secrets', namespace='eso', flags=['--create-namespace'], resource_deps=['vault'], labels=['eso'])

# Configure ESO to authenticate with Vault
local_resource(
  name='enable-k8s-auth',
  cmd="""vault auth enable kubernetes

k8s_host="$(kubectl exec vault-0 -n vault -- printenv | grep KUBERNETES_PORT_443_TCP_ADDR | cut -f 2- -d "=" | tr -d " ")"
k8s_port="443"            
k8s_cacert="$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 --decode)"
secret_name="$(kubectl get serviceaccount vault -n vault -o jsonpath='{.secrets[0].name}')"
tr_account_token="$(kubectl get secret ${secret_name} -n vault -o jsonpath='{.data.token}' | base64 --decode)"

vault write auth/kubernetes/config token_reviewer_jwt="${tr_account_token}" kubernetes_host="https://${k8s_host}:${k8s_port}" kubernetes_ca_cert="${k8s_cacert}" 
disable_issuer_verification=true""",
  resource_deps=['eso'],
  auto_init=False, 
  trigger_mode=TRIGGER_MODE_MANUAL, 
  labels=['vault']
)

local_resource(
  name='create-policy',
  cmd="""vault policy write eso-policy -<<EOF     
path "kv/data/test-secret"                                                  
{  capabilities = ["read"]                
}                         
EOF""",
  resource_deps=['enable-k8s-auth'],
  labels=['eso']
)

local_resource(
  name='create-vault-role',
  cmd="""demo_secret_name="$(kubectl get serviceaccount external-secrets-operator -n eso -o jsonpath='{.secrets[0].name}')"
demo_account_token="$(kubectl get secret ${demo_secret_name} -n eso -o jsonpath='{.data.token}' | base64 --decode)"                  

vault write auth/kubernetes/role/eso-role \
    bound_service_account_names=external-secrets-operator \
    bound_service_account_namespaces=eso \
    policies=eso-policy \
    ttl=24h

vault write auth/kubernetes/login role=eso-role jwt=$demo_account_token iss=https://kubernetes.default.svc.cluster.local""",
  resource_deps=['enable-k8s-auth', 'create-policy'],
  labels=['eso']
)

k8s_yaml('secret-store.yaml')
k8s_resource(new_name='secret-store', objects=['vault-backend:ClusterSecretStore'], resource_deps=['external-secrets-operator', 'create-vault-role'], labels=['eso'])


# Create Secret in Vault
local_resource(
  name='enable-kv-storage',
  cmd='vault secrets enable -version=2 kv',
  resource_deps=['enable-k8s-auth'],
  labels=['vault']
)
