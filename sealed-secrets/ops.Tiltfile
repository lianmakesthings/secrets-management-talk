load('ext://helm_resource', 'helm_resource', 'helm_repo')

# Installing Bitnami Sealed Secrets
helm_repo('bitnami', 'https://bitnami-labs.github.io/sealed-secrets', labels=['sealed-secrets'])
helm_resource('sealed-secrets-controller', 'bitnami/sealed-secrets', namespace='sealed-secrets', flags=['--create-namespace'], resource_deps=['bitnami'], labels=['sealed-secrets'])
