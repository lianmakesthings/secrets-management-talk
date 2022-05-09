load('ext://namespace', 'namespace_create', 'namespace_inject')
allow_k8s_contexts('Default')
trigger_mode(TRIGGER_MODE_MANUAL)
ENV = os.getenv('ENV')

if ENV == "dev":
  include('./ops.Tiltfile')

# Create Secret in Vault
local_resource(
  name='add-vault-secret',
  cmd='vault kv put kv/test-secret ENV={} SOURCE=VAULT'.format(ENV),
  auto_init=False,
  resource_deps=['vault'],
  labels=['vault']
)

# Create External Secret
namespace_create('vault-eso-app')
k8s_yaml('external-secret.yaml')
k8s_resource(new_name='external-secret',
  objects=['vault-eso-app:namespace', 'vault-test-secret:externalsecret'],
  auto_init=False,
  labels=['eso']
)

# Read secret
k8s_yaml(namespace_inject(read_file('../app/pod-reading-secret.yaml'), 'vault-eso-app'))
k8s_resource(workload='reading-secret',
  auto_init=False,
  labels=['app']
)