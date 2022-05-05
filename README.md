configuration notes:
- need custom health checks for composite resource claims
- claim to composite resource relationship can be severed if prune is used without block list.

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
Note that `argo/argo-config/argocd-cm.yaml` is needed to detect composite resource claim health correctly.

Optional steps to access ArgoCD UI.
```bash
# get the admin user's password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# port-forward to ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Go to https://localhost:8080 then login with user name `admin` and password from the above command.

### install crossplane
WIP 

### install Vault

```
kubectl apply -f argo/vault/vault.yaml
```
Wait for the vault application in ArgoCD to become healthy.

#### configure vault
TODO: automate this

kubectl -n vault-system exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json

kubectl -n vault-system exec vault-0 -- vault operator unseal `cat cluster-keys.json | jq -r ".unseal_keys_b64[]"`

kubectl -n vault-system exec -it vault-0 -- /bin/sh
