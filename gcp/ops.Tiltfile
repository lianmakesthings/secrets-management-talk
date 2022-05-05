load('ext://helm_resource', 'helm_resource', 'helm_repo')

# Installing External Secrets Operator
helm_repo('eso', 'https://charts.external-secrets.io', labels=['eso'])
helm_resource('external-secrets-operator', 'eso/external-secrets', namespace='eso', flags=['--create-namespace'], labels=['eso'])

# # Allow ESO to retrieve GCP Secrets
# local_resource('attach-SA-to-GCP',
#   cmd='gcloud iam service-accounts add-iam-policy-binding external-secrets@secret-management-talk.iam.gserviceaccount.com \
#     --role roles/iam.workloadIdentityUser \
#     --member "serviceAccount:secret-management-talk.svc.id.goog[eso/external-secrets]"',
#     labels=['gcp']
# )
k8s_yaml('gcp-sa-key-secret.yaml')
k8s_yaml('secret-store.yaml')
k8s_resource(new_name='gcp-secret-store', objects=['gcpsm-secret:Secret:eso', 'gcp-backend:ClusterSecretStore'], resource_deps=['external-secrets-operator'], labels=['gcp'])

# # Create GCP Secrets
# local_resource(
#   name='create-gcp-secrets',
#   cmd="""printf "PROD" | gcloud secrets create ENV --data-file=-
# printf "GCP" | gcloud secrets create SOURCE --data-file=-""",
#   labels=['gcp']
# )