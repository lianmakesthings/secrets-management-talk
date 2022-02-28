# Setup
- Secret `test-secret` FOO: BAR
- Pod which reads the secret and provides it via the environment

# Case 1
## Objective
- Secret provided as Sealed Secret
- Secret managed by team directly

## Usage
### Seal Secret for remote cluster usage
- Install sealed secrets controller on remote cluster via helm
```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets
```
- Install kubeseal, fetch pubkey from cluster
```
brew install kubeseal
kubeseal --fetch-cert --controller-name=sealed-secrets --controller-namespace=default > remote-cluster/pub-cert.pem
```
- Seal Secret with kubeseal
```
kubeseal --cert=remote-cluster/pub-cert.pem --format=yaml < k8s/k8s-secret.yaml > k8s/sealed-secret.yaml
```
### make sealed secret usable on local cluster
- Fetch private keys from remote cluster
```
kubectl get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > remote-cluster/master.key
```
- Install sealed secrets on local cluster (done via tilt)
- Update private keys on local cluster's sealed secrets controller
```
kubectl apply -f remote-cluster/master.key
kubectl delete pod -l name=sealed-secrets
```

# Case 2
- Secret provided by External Secret Operator
- Secret managed by Vault