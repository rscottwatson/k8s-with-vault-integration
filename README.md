# PURPOSE
This is a setup of using vault secret injection into pods.  The goal of this is to be able to better manage secrets stored in vault and how we get them into our pods.  This will also help with secert rotation as the pods should restart once the secret has changed.

This will compare two different options
1) agent injector
2) CSI


## Issues
Since the minikube cluster does not expose the host.minikube.internal name to the resources running in the cluster, I had to either add the following to the pod/deployment definition or change the VAULT URL to the IP address of my host gateway so that I could access the vault server runing in my terminal session.

      hostAliases:
      - hostnames:
          - "host.minikube.internal"
        ip: "192.168.79.1"

You will need to update the ip to your gateway IP address. 
TODO: Change this to use kustomize?

Minikube kept setting my gateway to 192.168.68.1 yet that was not the IP address of my bridge network on my host so I had to make sure it was set properly each time I started minikube.

## prerequisite
1) [minikube](https://gist.github.com/rscottwatson/e0e3c890b3d4aa81e46bf2993e3e216f)
2) [setup of vault agent sidecar injector](https://www.vaultproject.io/docs/platform/k8s/injector/installation)
3) [setup for CSI driver](https://www.vaultproject.io/docs/platform/k8s/csi/installation)
4) A vault running in dev mode ( I am using version v1.10 )
5) helm ( version v3.8.1 )


## Notes
To restart a pod when a secret has changed we can use the vault-hashicorp-com-agent-inject-command to kill the pod and restart it.
See this [demo](https://github.com/jasonodonnell/vault-agent-demo/blob/8900638491135bcf72aad3fc59022c4cf371f47a/examples/injector/dynamic-secrets/patch-annotations.yaml#L15) 
- [issue](https://github.com/hashicorp/vault-k8s/issues/196)
- Someone said to create another container that watches for OS changes and then will do a restart of the pod or deployment.  Not quite sure how to implemet this.

There are some notes about how this doesn't work so well with istio.  This would need some investigation as istio is a pretty important service mesh.
- [issue](https://github.com/hashicorp/vault-k8s/issues/41)


## References
[Installing vault with helm](https://www.vaultproject.io/docs/platform/k8s/helm/run)
  [helm configuration](https://www.vaultproject.io/docs/platform/k8s/helm/configuration)
[Injecting Secrets into Kubernetes Pods via Vault Agent Containers](https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar)
[Vault agent with kubernetes](https://learn.hashicorp.com/tutorials/vault/agent-kubernetes)
[Agent Injector vs Vault CSI Proiver](https://www.vaultproject.io/docs/platform/k8s/injector-csi)
[Installing the Secret Store CSI driver](https://secrets-store-csi-driver.sigs.k8s.io/getting-started/installation.html)
- [Azure](https://azure.github.io/secrets-store-csi-driver-provider-azure/)
- [Vault](https://github.com/hashicorp/secrets-store-csi-driver-provider-vault)
[Secret Provider Class Configuration](https://www.vaultproject.io/docs/platform/k8s/csi/configurations#secret-provider-class-configurations)