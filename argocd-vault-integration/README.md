# ArgoCD Vault Integration
---
# What?
This effort is to show we can have ArgoCD as our deployment management tool on Kubernetes integrated with Vault as a secrets / credentials management tool.
___
# Why?
Considering ArgoCD as a tool for orchestrating and managing deployments through *GitOps*, It's crucial to have a secure way for managing *Secrets* as we tend to store sensitive data in a secure way (not plain text or even more fancier, base64 encoded format) in the git repository. As we have *Vault* as a tool by HashiCorp for securely managing secrets, credentials, and sensitive data, offering encryption, access control, and auditing features, integrating ArgoCD with this tool can adequately satisfy all security requirements we have.

---
# How?
We will try to cover the above mentioned following these steps:
 - Installing Vault via ArgoCD
 - Creating required service account, clusterRole and clusterRoleBinding required for argoCD-vault integration
  - Configure Vault for being authenticated interacting with Kubernetes
   - Install AVP plugin on ArgoCD and adding the configuration
   - Creating secrets object with data stored in Vault

NOTE: The full description on each steps can be found in the video as well. This README is only for keeping the codes with a short description of each part.   

### Installing Vault via ArgoCD

We need to have both ArgoCD and Vault up and running. for the purpose of tutorial, We use *Kind* for bringing up Kubernetes clusters. Required Kind clusters configuration can be found under **infrastructure** directory. Following commands are being used to bring up the clusters:

```
cd infrastructure
kind create cluster --name lab-argocd --config ./ArgoCD-cluster-config.yaml
kind create cluster --name lab-vault --config ./Vault-cluster-config.yaml
```
**NOTE: Do not forget to replace your interface IP address to config files**

We shall have two kubernetes cluster up and running. One will be used only for hosting ArgoCD and the other for hosting Vault.

Change the context to ArgoCD cluster and then install ArgoCD via Helm

```
kubectx kind-lab-argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create ns argocd
helm install argocd argo/argo-cd -n argocd
```
If not via ingress, You can always use `kubectl port-forward ...` to access to the UI. Consider following as an example:
```
kubectl port-forward service/argocd-server -n argocd --address 0.0.0.0 8080:443
``` 
NOTE: use `admin` as username. Password can be found via following command
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Following the steps, You will have fresh instance of ArgoCD up and running.

### Adding Vault cluster to ArgoCD and deploying Vault
Login to ArgoCD via `argocd cli` and add `kind-lab-vault` cluster. 
```
argocd login 127.0.0.1:8080
argocd cluster add kind-lab-vault
``` 
Next we need add this git repository to ArgoCD attached repositories. We do that via argoCD UI. ***All applications that are supposed to be deployed by ArgoCD, including Vault, will be placed under applications directory of this repository***

Having the repository added to the ArgoCD repositories, Create a ArgoCD application. The target should be this repository and path should be ***argocd-vault-integration/applications***. Set the type of the application to ***directory*** and keep ***recursive*** enabled.

If all the configs set correctly, you will see the vault application being created and coming up.

**This is important to note vault by default is coming up sealed**. Meaning the pod will not be ready until you make vault unsealed. Following is how to
```
kubectx kind-lab-vault
kubens vault
kubectl exec -it vault-0 sh
vault operator init
```
This will provide you with **Initial root Token** along with 5 unseal keys. Keep this information somewhen as they will be required later. 3 out of 5 Unseal keys should be used to unseal the vault using command `vault operator unseal <KEY>`.  