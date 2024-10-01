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