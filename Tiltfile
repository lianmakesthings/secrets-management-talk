load('ext://helm_resource', 'helm_resource', 'helm_repo')

# Installing Bitnami Sealed Secrets
# helm_repo('bitnami', 'https://bitnami-labs.github.io/sealed-secrets')
# helm_resource('sealed-secrets', 'bitnami/sealed-secrets')
# Applying sealed secret
# k8s_yaml('sealed-secrets/sealed-secret.yaml')
# k8s_resource(workload='sealed-secret', labels=['app'])

# Installing External Secrets Operator
helm_repo('eso', 'https://charts.external-secrets.io', labels=['eso'])
helm_resource('external-secrets', 'eso/external-secrets', namespace='eso', flags=['--create-namespace'], labels=['eso'])

# # Allow ESO to retrieve GCP Secrets
# local_resource('attach-SA-to-GCP',
#   cmd='gcloud iam service-accounts add-iam-policy-binding external-secrets@secret-management-talk.iam.gserviceaccount.com \
#     --role roles/iam.workloadIdentityUser \
#     --member "serviceAccount:secret-management-talk.svc.id.goog[eso/external-secrets]"',
#     labels=['gcp']
# )
# k8s_yaml('eso/gcp-sa-key-secret.yaml')
# k8s_yaml('eso/gcp-secret-store.yaml')
# k8s_resource(new_name='gcp-secret', objects=['gcpsm-secret:secret', 'secret-mgmt-talk:secretstore'], labels=['gcp'])

# # Create GCP Secrets
# local_resource(
#   name='create-gcp-secrets',
#   cmd="""printf "BAR" | gcloud secrets create FOO --data-file=-
# printf "GCP" | gcloud secrets create SOURCE --data-file=-""",
#   labels=['gcp']
# )

# # Create External Secret
# k8s_yaml('eso/external-secret.yaml')
# k8s_resource(new_name='gcp-external-secret', objects=['gcp-test-secret:externalsecret'], labels=['gcp'])

# Installing Hashicorp Vault
helm_repo('hashicorp', 'https://helm.releases.hashicorp.com', labels=['vault'])
helm_resource('consul', 'hashicorp/consul', namespace='vault', flags=['--create-namespace', '--values', 'vault/consul-values.yaml'], labels=['vault'])
helm_resource('vault', 'hashicorp/vault', namespace='vault', resource_deps=['consul'], flags=['--create-namespace', '--values', 'vault/vault-values.yaml'], labels=['vault'])
os.putenv('VAULT_ADDR', 'http://127.0.0.1:8200')
local_resource(
  name='forward-vault',
  serve_cmd='kubectl port-forward svc/vault -n vault 8200:8200',
  resource_deps=['vault'],
  labels=['vault']
)

# Configure ESO to authenticate with Vault
local_resource(
  name='create-policy',
  cmd="""vault policy write eso-policy -<<EOF     
path "kv/data/test-secret"                                                  
{  capabilities = ["read"]                
}                         
EOF""",
  resource_deps=['external-secrets', 'vault'],
  labels=['vault']
)
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
  resource_deps=['external-secrets', 'vault'],
  labels=['vault']
)
local_resource(
  name='create-vault-role',
  cmd="""demo_secret_name="$(kubectl get serviceaccount external-secrets -n eso -o jsonpath='{.secrets[0].name}')"
demo_account_token="$(kubectl get secret ${demo_secret_name} -n eso -o jsonpath='{.data.token}' | base64 --decode)"                  

vault write auth/kubernetes/role/eso-role \
    bound_service_account_names=external-secrets \
    bound_service_account_namespaces=eso \
    policies=eso-policy \
    ttl=24h

vault write auth/kubernetes/login role=eso-role jwt=$demo_account_token iss=https://kubernetes.default.svc.cluster.local""",
  resource_deps=['external-secrets', 'vault'],
  labels=['vault']
)

k8s_yaml('vault/secret-store.yaml')
k8s_resource(new_name='vault-secret-store', objects=['vault-backend:secretstore'], labels=['vault'])


# Create Secret in Vault
local_resource(
  name='add-vault-secret',
  cmd="""vault secrets enable -version=2 kv
vault kv put kv/test-secret FOO=BAR SOURCE=VAULT""",
  resource_deps=['external-secrets', 'vault'],
  labels=['vault']
)

# Create External Secret
k8s_yaml('vault/external-secret.yaml')
k8s_resource(new_name='vault-external-secret', objects=['vault-test-secret:externalsecret'], labels=['vault'])

k8s_yaml('app/pod-reading-secret.yaml')
k8s_resource(workload='reading-secret', labels=['app'])
