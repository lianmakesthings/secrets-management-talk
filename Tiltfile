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

# Allow ESO to retrieve GCP Secrets
local_resource('attach-SA-to-GCP',
  cmd='gcloud iam service-accounts add-iam-policy-binding external-secrets@secret-management-talk.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:secret-management-talk.svc.id.goog[eso/external-secrets]"',
    labels=['gcp']
)
k8s_yaml('eso/gcp-sa-key-secret.yaml')
k8s_yaml('eso/secret-store.yaml')
k8s_resource(new_name='gcp-secret', objects=['gcpsm-secret:secret', 'secret-mgmt-talk:secretstore'], labels=['gcp'])

k8s_yaml('eso/gcp-secret.yaml')
k8s_resource(new_name='external-test-secret', objects=['gcp-test-secret:externalsecret'], labels=['gcp'])

# Installing Hashicorp Vault
# helm_repo('hashicorp', 'https://helm.releases.hashicorp.com', labels=['vault'])
# helm_resource('consul', 'hashicorp/consul', namespace='vault', flags=['--create-namespace', '--values', 'vault/consul-values.yaml'], labels=['vault'])
# helm_resource('vault', 'hashicorp/vault', namespace='vault', resource_deps=['consul'], flags=['--create-namespace', '--values', 'vault/vault-values.yaml'], labels=['vault'])

# os.putenv('VAULT_ADDR', 'http://127.0.0.1:8200')
# local_resource(
#   name='forward-vault',
#   cmd='kubectl port-forward svc/vault -n vault 8200:8200',
#   resource_deps=['vault'],
#   labels=['vault']
# )

k8s_yaml('app/pod-reading-secret.yaml')
k8s_resource(workload='reading-secret', labels=['app'])
