# PURPOSE
This is a setup of using Vault secret injection into pods.  The goal of this is to be able to better manage secrets stored in vault and how we get them into our pods.  This will also help with secert rotation as the pods should restart once the secret has changed.  ( It turns out that restarting the pods is not as simple and needs more work)

During my research I also saw people mention that working with ISTIO was not as straight forward either.  This will need to be investigated further.

There are two solutions used to handle secrets from Vault.
1) agent injector
2) Secret Store CSI  


## Pre-requisites
1) [minikube](https://gist.github.com/rscottwatson/e0e3c890b3d4aa81e46bf2993e3e216f)
2) [setup of vault agent sidecar injector](https://www.vaultproject.io/docs/platform/k8s/injector/installation)
3) [setup for CSI driver](https://www.vaultproject.io/docs/platform/k8s/csi/installation)
4) A vault running in dev mode ( I am using version v1.10 )
5) helm ( version v3.8.1 )


## Notes
To restart a pod when a secret has changed we can use the vault-hashicorp-com-agent-inject-command annotation to kill the pod and restart it. (Agent Inject only not the CSI driver)
See this [demo](https://github.com/jasonodonnell/vault-agent-demo/blob/8900638491135bcf72aad3fc59022c4cf371f47a/examples/injector/dynamic-secrets/patch-annotations.yaml#L15) 
- [issue about restarting a pod on secret update](https://github.com/hashicorp/vault-k8s/issues/196)
- Someone said to create another container that watches for OS changes and then do a restart of the pod or deployment.  Not quite sure how to implemet this.  Maybe we can use the (inotify or systemd path units)[https://www.linuxjournal.com/content/linux-filesystem-events-inotify]?
- Here is an example of fswatch which used inotify. (link)[https://william-yeh.net/post/2019/06/inotify-in-containers/] 

There are some notes about how this doesn't work so well with istio.  This would need some investigation as istio is a pretty important service mesh.
- [issue with istio and agent injector](https://github.com/hashicorp/vault-k8s/issues/41)

## Issues
Since the minikube cluster does not resolve the /etc/hosts alias host.minikube.internal from within PODS running in the cluster, I had to either add the following to the pod/deployment definition or change the VAULT URL to the IP address of my host gateway so that I could access the Vault server runing in my terminal session on my desktop.

      hostAliases:
      - hostnames:
          - "host.minikube.internal"
        ip: "192.168.79.1"

You will need to update the ip to your gateway IP address. 
TODO: Change this to use kustomize?

Minikube kept setting my /etc/hosts entry for host.minikube.internal to 192.168.68.1 yet that was not the IP address of my bridge network on my host so I had to make sure it was set properly each time I started minikube.
## References
- [Installing Vault with Helm](https://www.vaultproject.io/docs/platform/k8s/helm/run)
  - [Helm configuration](https://www.vaultproject.io/docs/platform/k8s/helm/configuration)
- [Injecting Secrets into Kubernetes Pods via Vault agent sidecar containers](https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar)
- [Vault agent with Kubernetes](https://learn.hashicorp.com/tutorials/vault/agent-kubernetes)
- [Agent Injector vs Vault CSI Proiver](https://www.vaultproject.io/docs/platform/k8s/injector-csi)
- [Installing the Secret Store CSI driver](https://secrets-store-csi-driver.sigs.k8s.io/getting-started/installation.html)
  - [Azure key vault](https://azure.github.io/secrets-store-csi-driver-provider-azure/)
  - [Vault](https://github.com/hashicorp/secrets-store-csi-driver-provider-vault)
- [Secret Provider Class Configuration](https://www.vaultproject.io/docs/platform/k8s/csi/configurations#secret-provider-class-configurations)