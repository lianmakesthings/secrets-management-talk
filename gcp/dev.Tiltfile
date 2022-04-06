load('ext://namespace', 'namespace_create', 'namespace_inject')

# Create External Secret
k8s_yaml('external-secret.yaml')
k8s_resource(new_name='external-secret', objects=['gcp-test-secret:externalsecret'], labels=['app'])

# Read secret
namespace_create('gcp-eso-app')
k8s_yaml(namespace_inject(read_file('../app/pod-reading-secret.yaml'), 'gcp-eso-app'))
k8s_resource(new_name='app', workload='reading-secret', objects=['gcp-eso-app:namespace'], labels=['app'])