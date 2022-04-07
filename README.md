# PURPOSE
This is a setup of using vault secret injection into pods.  The goal of this is to be able to better manage secrets stored in vault and how we get them into our pods.  This will also help with secert rotation as the pods should restart once the secret has changed.

This will compare two different options
1) agent sidecar injector
2) CSI

## prerequisite

1) [minikube](https://gist.github.com/rscottwatson/e0e3c890b3d4aa81e46bf2993e3e216f)
2) [setup of vault agent sidecar injector](https://www.vaultproject.io/docs/platform/k8s/injector/installation)
3) A vault running in dev mode ( I am using version v1.10 )
4) helm ( version v3.8.1 )

## setup

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
minikube start --driver=hyperkit -p vaultAgent

# Set the default profile so we don't need to keep supplying it.
minikube config set profile vaultAgent
```


Use helm to export a template of the yaml used to deploy the kubernetes resources for the agent injector.
Using helm is an options but I want to see if I can eventually use ArgoCD to deploy this with Kustomize which is why I am not using helm.  Plus I am not a fan of helm with no real reason other than I don't feel this is the way forward.
TODO

```bash
# make sure we are pointing to correct cluster
kubectl config current-context

helm repo add hashicorp https://helm.releases.hashicorp.com
helm search repo hashicorp/vault
helm install vault hashicorp/vault --set="injector.enabled=true" \
                                   --set "injector.externalVaultAddr=http://host.minikube.internal:8200"

# This is the output from the install command
# ===========================================
# NAME: vault
# LAST DEPLOYED: Tue Apr  5 09:46:44 2022
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1
# NOTES:
# Thank you for installing HashiCorp Vault!

# Now that you have deployed Vault, you should look over the docs on using
# Vault with Kubernetes available here:

# https://www.vaultproject.io/docs/


# Your release is named vault. To learn more about the release, try:

#   $ helm status vault
#   $ helm get manifest vault


helm template \
    --set="injector.enabled=true" \
    --set "injector.externalVaultAddr=http://host.minikube.internal:8200" \
    hashicorp/vault > sidecar/vault_injector.yaml

```

Get the code from hashicorp.  This is to be used as a reference incase I need it later for now I have copied the only yaml out.
```bash
cd sidecar
git clone https://github.com/hashicorp/vault-guides.git
git clone https://github.com/hashicorp/learn-vault-agent.git
cd vault-guides/operations/provision-vault/kubernetes/minikube/vault-agent-sidecar

```


Create a secret in our external vault
```bash
vault kv put secret/database/mysql/dev username=admin password=not4you

```


Create a service account in Kubernetes that will allow our pods to access our external Vault server.
```bash
kubectl create serviceaccount vault-auth
kubectl apply -f ./sidecar/learn-vault-agent/vault-agent-k8s-demo/vault-auth-service-account.yaml

```

Create a policy that grants access to the secret in vault.
```
vault policy write myapp-kv-ro - <<EOF
path "secret/data/database/mysql/*" {
    capabilities = ["read", "list"]
}
EOF

```

Setup Vault to allow authentication with kubernetes auth provider.  Maybe look at using approle instead?  When using the k8s provider we need to do some initial setup as we need to let vault know about the Service Account (SA) that is going to be connecting.  Also k8s needs to be able to connect to vault and vault needs to be able to connect to k8s as well.
```bash
export VAULT_SA_NAME=$(kubectl get sa vault-auth \
                            --output jsonpath="{.secrets[*]['name']}")
export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME  \
                            --output 'go-template={{ .data.token }}' | base64 --decode)
export SA_CA_CRT=$(kubectl config view --raw --minify --flatten \
                            --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
export K8S_HOST=$(kubectl config view --raw --minify --flatten \
                            --output 'jsonpath={.clusters[].cluster.server}')

# could pass in the -path=k8s/vaultAgent  if we are going to have multiple clusters
# or if we have different clusters we would have to organize it like this
#  -path=/k8s/application/dev-eastus 
#  -path=/k8s/tooling/dev-eastus
#  -path=/k8s/application/prod-dewc
# etc 
vault auth enable -path=k8s/vaultAgent kubernetes  

# define the service account that will be allowed to connect to vault
vault write auth/k8s/vaultAgent/config \
    token_reviewer_jwt="$SA_JWT_TOKEN" \
    kubernetes_host="$K8S_HOST" \
    kubernetes_ca_cert="$SA_CA_CRT" \
    issuer="https://kubernetes.default.svc.cluster.local"

```

Create a vault role to map the k8s service account name to the vault policy this corresponds to.
```bash
vault write auth/k8s/vaultAgent/role/example \
     bound_service_account_names=vault-auth \
     bound_service_account_namespaces=default \
     policies=myapp-kv-ro \
     ttl=24h

```

Quick check to make sure vault is accessible from our minikube cluster
```
minikube ssh
curl -s http://host.minikube.internal:8200/v1/sys/seal-status
```



Create a POD that will refernece the IP address of the host.minikube.internal address.

```
# need to check that this doesn't have some character between the IP and hostname
EXTERNAL_VAULT_ADDR=$(minikube ssh "cat /etc/hosts | grep host.minikube.internal | cut -d$'\t' -f1")


cat > devwebapp.yaml << EOF 
apiVersion: v1
kind: Pod
metadata:
  name: devwebapp
  labels:
    app: devwebapp
spec:
  serviceAccountName: vault-auth
  containers:
    - name: devwebapp
      image: burtlo/devwebapp-ruby:k8s
      env:
        - name: VAULT_ADDR
          value: "http://$EXTERNAL_VAULT_ADDR:8200"
EOF

kubectl apply -f devwebapp.yaml
```

Once the pod becomes ready open a shell to it to test that we can access our vault

```bash
kubectl exec --stdin=true --tty=true devwebapp -- /bin/sh

#The pod is running with the service account we created earlier.
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

#We can use this token to login to our vault
curl --request POST \
       --data '{"jwt": "'"$KUBE_TOKEN"'", "role": "example"}' \
       $VAULT_ADDR/v1/auth/k8s/vaultAgent/login

#works with wget too
wget -qO - --post-data '{"jwt": "'"$KUBE_TOKEN"'", "role": "example"}' $VAULT_ADDR/v1/auth/k8s/vaultAgent/login

# review the output and exit the pod
exit
```


Enable auto-auth for the pod and get a secret automatically.
```bash
# create the configmap that contains the vault config information.
kubectl apply -f ./sidecar/example-vault-agent-configmap.yaml

# deploy pod that will automatically get a secret from vault.
kubectl apply -f ./sidecar/vault-agent-example.yaml

# check that we can get the secret information from the pod.  This is using the template defined in the configmap to
# generate the output in a format that can be used by the pod.
minikube ssh "curl $(kubectl get pod vault-agent-example --output jsonpath={.status.podIP})"
```


Run some other examples
## using annotations
```bash
# This will run a pod with an init container and another pod to watch for changes from vault. 
kubectl apply -f ./sidecar/example-annotation.yaml  

kubectl exec deploy/app-example-deployment  -- cat /vault/secrets/db-creds 
# or add the continer name if the app isn't the default container
kubectl exec deploy/app-example-deployment -c app  -- cat /vault/secrets/db-creds 

#get the pod information
kubectl get pod -l app=app-example  --output jsonpath='{range .items[].status.containerStatuses[*]}{@.name}{"  "}{@.state}{"\n"}{end}'

#now update the secret
vault kv put secret/database/mysql/dev username=admin password=not4youversion2

#wait a few minutes and run
kubectl exec deploy/app-example-deployment -c app  -- cat /vault/secrets/db-creds 
# and you should see the secret updated.

#notice how the pod did not restart
kubectl get pod -l app=app-example  --output jsonpath='{range .items[].status.containerStatuses[*]}{@.name}{"  "}{@.state}{"\n"}{end}'

#Question should the POD restart when there is a change to the secret?  Can it?
```

## using environment variables
```bash
# This will run a pod with an init container and another pod to watch for changes from vault. 
kubectl apply -f ./sidecar/example-envvar.yaml

kubectl logs -l app=web -c web -f

# change the value of the secret in vault.  ie using the ui for example.

```


## clean up the environment
```bash
# Rmove all the deployments and pods
kubectl delete -f ./sidecar/vault-agent-example.yaml

kubectl delete -f ./sidecar/example-vault-agent-configmap.yaml

kubectl delete -f ./sidecar/example-annotation.yaml  

kubectl delete -f ./sidecar/example-envvar.yaml


```


## Notes
To restart a pod when a secret has changed we can use the vault-hashicorp-com-agent-inject-command to kill the pod and restart it.
See this [demo](https://github.com/jasonodonnell/vault-agent-demo/blob/8900638491135bcf72aad3fc59022c4cf371f47a/examples/injector/dynamic-secrets/patch-annotations.yaml#L15) 
- [issue](https://github.com/hashicorp/vault-k8s/issues/196)
- Someone said to create another container that watches for OS changes and then will do a restart of the pod or deployment.  Not quite sure how to implemet this.

There are some notes about how this doesn't work so well with istio.  This would need some investigation as istio is a pretty important service mesh.
- [issue](https://github.com/hashicorp/vault-k8s/issues/41)


## references
[Installing vault with helm](https://www.vaultproject.io/docs/platform/k8s/helm/run)
[Injecting Secrets into Kubernetes Pods via Vault Agent Containers](https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar)
[Vault agent with kubernetes](https://learn.hashicorp.com/tutorials/vault/agent-kubernetes)
[Installing the Secret Store CSI driver](https://secrets-store-csi-driver.sigs.k8s.io/getting-started/installation.html)
- [Azure](https://azure.github.io/secrets-store-csi-driver-provider-azure/)
- [Vault](https://github.com/hashicorp/secrets-store-csi-driver-provider-vault)