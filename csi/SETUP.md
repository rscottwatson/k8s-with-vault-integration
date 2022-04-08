# CSI

## Setup

Start an external vault as our source of secrets.  I don't want to use vault running inside of kubernetes as my use case will be to use our corporate vault service.

In a shell window run the following.  Note this is using root as the token value

``` bash
# need to listen on 0.0.0.0 to allow minikube to talk to the server
vault server -dev  -dev-listen-address=0.0.0.0:8200 -dev-root-token-id root

# if you want to access the UI use the following command
open http://127.0.0.1:8200/ui 

# to connect to vault on the command line use the following
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN=root

# make sure you can run some vault commands
vault status
vault secrets list

wget -qO - ${VAULT_ADDR}/v1/sys/health
curl ${VAULT_ADDR}/v1/sys/health

```



Setup a new minikube instance to play with

```bash
minikube start --driver=hyperkit -p vaultCSI

# Set the default profile so we don't need to keep supplying it.
minikube config set profile vaultCSI

```

Install the Secret Store CSI Driver and the VAULT CSI provider
```bash

# check we are connected to the right cluster
kubectl cluster-info 

# With HELM3
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo update secrets-store-csi-driver
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system \
     --set "syncSecret.enabled=true" --set "enableSecretRotation=true"

#Options 
# Sync as Kubernetes secret      syncSecret.enabled=true
# Secret Auto rotation           enableSecretRotation=true


# With YAML
kubectl apply -f deploy/rbac-secretproviderclass.yaml
kubectl apply -f deploy/csidriver.yaml
kubectl apply -f deploy/secrets-store.csi.x-k8s.io_secretproviderclasses.yaml
kubectl apply -f deploy/secrets-store.csi.x-k8s.io_secretproviderclasspodstatuses.yaml
kubectl apply -f deploy/secrets-store-csi-driver.yaml

# If using the driver to sync secrets-store content as Kubernetes Secrets, deploy the additional RBAC permissions
# required to enable this feature
kubectl apply -f deploy/rbac-secretprovidersyncing.yaml

# If using the secret rotation feature, deploy the additional RBAC permissions
# required to enable this feature
kubectl apply -f deploy/rbac-secretproviderrotation.yaml

# Install Vault CSI Provider ( only as I will use the external vault )
helm repo add hashicorp https://helm.releases.hashicorp.com
helm search repo hashicorp/vault
helm install vault hashicorp/vault \
    --set "injector.enabled=false" \
    --set "csi.enabled=true" \
    --set "server.enabled=false"
```


## Setup our secrets
```bash
vault kv put secret/database/mysql/dev username=admin password=not4you
```

## Setup our service account that will be used to auth to k8s
```
kubectl apply -f ./csi/myapp-csi-sa.yaml 
```

## Setup Vault authentication
```bash
export VAULT_SA_NAME=$(kubectl get sa myapp-csi-sa \
                            --output jsonpath="{.secrets[*]['name']}")
export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME  \
                            --output 'go-template={{ .data.token }}' | base64 --decode)
export SA_CA_CRT=$(kubectl config view --raw --minify --flatten \
                            --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
export K8S_HOST=$(kubectl config view --raw --minify --flatten \
                            --output 'jsonpath={.clusters[].cluster.server}')

vault auth enable -path=k8s/vaultCSI kubernetes  

# define the service account that will be allowed to connect to vault
vault write auth/k8s/vaultCSI/config \
    token_reviewer_jwt="$SA_JWT_TOKEN" \
    kubernetes_host="$K8S_HOST" \
    kubernetes_ca_cert="$SA_CA_CRT" \
    issuer="https://kubernetes.default.svc.cluster.local"

```

## Setup Vault authorization

```
vault policy write myapp-kv-ro - <<EOF
path "secret/data/database/mysql/*" {
    capabilities = ["read", "list"]
}
EOF

```

## Tie the authentication together with the authorization

Create a vault role to map the k8s service account name to the vault policy this corresponds to.

```bash
vault write auth/k8s/vaultCSI/role/example \
     bound_service_account_names=myapp-csi-sa \
     bound_service_account_namespaces=default \
     policies=myapp-kv-ro \
     ttl=24h

```


## Examples
1. kubectl apply -f ./csi/vault-example1.yaml 






## Errors
```
cannot create resource \"tokenreviews\" in API group \"authentication.k8s.io\" at the cluster scope","reason":"Forbidden","details":{"group":"authentication.k8s.io","kind":"tokenreviews"},"code":403}

Was becasue I forgot to create the role binding granting my service account access to the cluster role auth-delegator.

kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vmyapp-csi-sa
  namespace: default

```