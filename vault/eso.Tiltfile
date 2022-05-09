load('ext://helm_resource', 'helm_resource', 'helm_repo')
allow_k8s_contexts('Default')

helm_repo('eso', 'https://charts.external-secrets.io', labels=['eso'])
helm_resource('external-secrets-operator', 'eso/external-secrets', namespace='eso', flags=['--create-namespace'], resource_deps=['vault'], labels=['eso'])
