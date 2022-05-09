load('ext://namespace', 'namespace_create', 'namespace_inject')
allow_k8s_contexts('Default')
ENV = os.getenv('ENV')
trigger_mode(TRIGGER_MODE_MANUAL)

if ENV == "dev":
  include('./ops.Tiltfile')

# Fetch certkey from cluster
local_resource('fetch-cert',
  cmd='kubeseal --fetch-cert --controller-name=sealed-secrets-controller --controller-namespace=sealed-secrets > pub-cert.pem',
  auto_init=False,
  labels=['sealed-secrets']
)

# Seal secret
local_resource('seal-secret',
  cmd='kubeseal --cert=pub-cert.pem --format=yaml < {}-k8s-secret.yaml > sealed-secret.yaml'.format(ENV),
  resource_deps=['fetch-cert'],
  auto_init=False,
  labels=['sealed-secrets']
)

# Deploying sealed secret
namespace_create('sealed-secret-app')
k8s_yaml('sealed-secret.yaml')
k8s_resource(new_name='sealed-secret',
  objects=['sealed-secret-app:namespace', 'test-secret:SealedSecret'],
  resource_deps=['seal-secret'],
  auto_init=False,
  labels=['app']
)

# Read secret
k8s_yaml(namespace_inject(read_file('../app/pod-reading-secret.yaml'), 'sealed-secret-app'))
k8s_resource(new_name='app',
  workload='reading-secret',
  resource_deps=['seal-secret'],
  auto_init=False,
  labels=['app'])