# Installing Bitnami Sealed Secrets

load('ext://helm_resource', 'helm_resource', 'helm_repo')
helm_repo('bitnami', 'https://bitnami-labs.github.io/sealed-secrets')
helm_resource('sealed-secrets', 'sealed-secrets/sealed-secrets')

k8s_yaml('k8s/sealed-secret.yaml')
#k8s_resource(workload="test-secret", labels=["secrets"])

k8s_yaml('k8s/pod-reading-secret.yaml')
