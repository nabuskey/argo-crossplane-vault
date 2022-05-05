```
kubectl apply -f vault.yaml

kubectl -n vault-system exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json

kubectl -n vault-system exec vault-0 -- vault operator unseal `cat cluster-keys.json | jq -r ".unseal_keys_b64[]"`

kubectl -n vault-system exec -it vault-0 -- /bin/sh

vault login # use root token from above

vault auth enable kubernetes
vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        
exit # exit vault container

kubectl -n vault-system exec -it vault-0 -- vault secrets enable -path=secret kv-v2

kubectl -n vault-system exec -i vault-0 -- vault policy write crossplane - <<EOF
path "secret/data/cro*" {
    capabilities = ["create", "read", "update", "delete"]
}
path "secret/metadata/*" {
    capabilities = ["create", "read", "update", "delete"]
}
EOF

kubectl -n vault-system exec -it vault-0 -- vault write auth/kubernetes/role/crossplane \
    bound_service_account_names="*" \
    bound_service_account_namespaces=crossplane-system \
    policies=crossplane \
    ttl=24h

kubectl apply -f storeconfig.yaml



# application stuff

ubectl -n vault-system exec -i vault-0 -- sh
# in vault conainer
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



