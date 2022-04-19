load('ext://namespace', 'namespace_create', 'namespace_inject')

# Create External Secret
k8s_yaml('external-secret.yaml')
k8s_resource(new_name='external-secret', objects=['vault-test-secret:externalsecret'], labels=['eso'])

# Read secret
namespace_create('vault-eso-app')
k8s_yaml(namespace_inject(read_file('../app/pod-reading-secret.yaml'), 'vault-eso-app'))
k8s_resource(workload='reading-secret', objects=['vault-eso-app:namespace'], labels=['app'])