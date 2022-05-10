This guide covers three objectives:
1. Publish Crossplane secrets to Vault.
2. Deploy an example app which uses Crossplane through ArgoCD.
3. Allow the application to have access to resources created by Crossplane. 

Note:
 - This guide uses in-cluster ArgoCD and Vault but they can be external. They are in the same cluster for ease of use.
 - Requires Crossplane `1.7.0+` and AWS provider `0.26.0+`

## Installation Steps

### Create a cluster
WIP

### Install ArgoCD
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl apply -f argo/argo-config
```
Note:
    - `argo/argo-config/argocd-cm.yaml` is needed to detect composite resource claim health correctly.
    - `argo/argo-config/project.yaml` is needed to prevent Argo from severing ties between Composite Resource Claim and Composite Resource when pruning is enabled in Argo.

#### Optional steps to access ArgoCD UI.
```bash
# get the admin user's password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# port-forward to ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Go to https://localhost:8080 then login with user name `admin` and password from the above command.

### install Vault

```bash
kubectl apply -f argo/vault/vault.yaml
```
Wait for the vault application in ArgoCD to become healthy.

#### configure vault
TODO: automate this

Based on [this guide](https://github.com/crossplane/crossplane/blob/master/docs/guides/vault-as-secret-store.md) from Crossplane repository. See the guide if you want to know more about this process.

```bash 
kubectl -n vault-system exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json

kubectl -n vault-system exec vault-0 -- vault operator unseal `jq -r ".unseal_keys_b64[]" cluster-keys.json`

# get root token
jq -r ".root_token" cluster-keys.json
kubectl -n vault-system exec -it vault-0 -- /bin/sh
# commands below are executed in vault container.  login using root token form above.
vault login 
vault auth enable kubernetes
vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
vault secrets enable -path=secret kv-v2

# create policy and role for Crossplane to use.
vault policy write crossplane - <<EOF
path "secret/data/cro*" {
    capabilities = ["create", "read", "update", "delete"]
}
path "secret/metadata/*" {
    capabilities = ["create", "read", "update", "delete"]
}
EOF

vault write auth/kubernetes/role/crossplane \
    bound_service_account_names="*" \
    bound_service_account_namespaces=crossplane-system \
    policies=crossplane \
    ttl=24h

# create policy and role for applications to use.
ACCESSOR=$(vault auth list | grep kubernetes | tr -s ' ' | cut -d ' ' -f3)

vault policy write k8s-application - << EOF
path "secret/data/crossplane-system/{{identity.entity.aliases.${ACCESSOR}.metadata.service_account_namespace}}/*" {
  capabilities = ["read", "list"]
}
path "secret/metadata/crossplane-system/{{identity.entity.aliases.${ACCESSOR}.metadata.service_account_namespace}}/*" {
  capabilities = ["read", "list"]
}
EOF

vault write auth/kubernetes/role/k8s-application \
    bound_service_account_names="*" \
    bound_service_account_namespaces=* \
    policies=k8s-application \
    ttl=1h
```

### install crossplane

```bash
kubectl create ns crossplane-system
helm upgrade --install crossplane --namespace crossplane-system crossplane-stable/crossplane --version 1.7.0 --values crossplane/values.yaml


kubectl apply -f crossplane/providers/aws/aws-provider.yaml
kubectl apply -f crossplane/providers/aws/aws-jet-provider.yaml

kubectl apply -f crossplane/storeconfig.yaml

```
