load('ext://namespace', 'namespace_create', 'namespace_inject')

# Fetch certkey from cluster
local_resource('fetch-cert',
  cmd='kubeseal --fetch-cert --controller-name=sealed-secrets-controller --controller-namespace=sealed-secrets > pub-cert.pem',
  labels=['sealed-secrets']
)

# Seal secret
local_resource('seal-secret',
  cmd='kubeseal --cert=pub-cert.pem --format=yaml < k8s-secret.yaml > sealed-secret.yaml',
  resource_deps=['fetch-cert'],
  labels=['sealed-secrets']
)

# Deploying sealed secret
k8s_yaml('sealed-secret.yaml')
k8s_resource(new_name='sealed-secret', objects=['test-secret:SealedSecret'], resource_deps=['seal-secret'], labels=['app'])

# Read secret
namespace_create('sealed-secret-app')
k8s_yaml(namespace_inject(read_file('../app/pod-reading-secret.yaml'), 'sealed-secret-app'))
k8s_resource(new_name='app', workload='reading-secret', objects=['sealed-secret-app:namespace'], resource_deps=['seal-secret'],labels=['app'])