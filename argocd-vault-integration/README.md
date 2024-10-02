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
This will provide you with **Initial root Token** along with 5 unseal keys. Keep this information somewhen as they will be required later. 3 out of 5 Unseal keys should be used to unseal the vault using command `vault operator unseal <KEY>`. Lets say following is the output
```
Unseal Key 1: oKDDHikSmx0XwX8V5zgy0CCH/Mle/gz7synSn0SJ+/tl
Unseal Key 2: hXxYj575680eTSas1C0TQJOwxPKTCDBxB1xfvkySv97z
Unseal Key 3: 8LpgJGgrvJuXWRw17S6XF0c2GywD4+cUtdQ2KdeoQQYZ
Unseal Key 4: VHp14753wUwsjthkF9YJR7F+Ny5besCN0YdBVHuoI8UI
Unseal Key 5: qg0Yg2pPD/QQq5tR8znqiLh0Axz1Ayxak/WTqiUr8fTX

Initial Root Token: hvs.4IhgBifbmiFifYK7HC0PmJ0d

```

Then following command will make vault unsealed and accessible
```
vault operator unseal hXxYj575680eTSas1C0TQJOwxPKTCDBxB1xfvkySv97z
vault operator unseal oKDDHikSmx0XwX8V5zgy0CCH/Mle/gz7synSn0SJ+/tl
vault operator unseal 8LpgJGgrvJuXWRw17S6XF0c2GywD4+cUtdQ2KdeoQQYZ
```

If not using Ingress, Expose Vault UI using following command and login using your initial root token 
```
kubectl port-forward service/vault --address 0.0.0.0   8200:8200
```
**NOTE: It is important to expose the port on all interfacess using --address argument. Later ArgoCD needs to connect to vault, which will be through interface IP address**

Create new secret with the path `development/sample-secret` with a key `lab-secret` and a desired value.
```
Secrets Engine > Enable new Engine > KV > Enable Engine > create secret 
```
Add a `Policy` with name `development` to define the path / secret accessibility 
```
Policies > Create ACL policy
```
Set to ACL rule as follows
```
# Manage auth methods broadly across Vault
path "development/*"
{
  capabilities = [ "read", "list"]
}
```

This policy will provide access to secrets stored in the path `development` for any authenticated entity that uses this policy.

Having everything in place, its time to setup `Kubernetes` configuration. This will enable enable entities from inside the Kubernetes cluster to be authenticated by Vault. In out case, ArgoCD requests to Vault, originates from inside Kubernetes cluster that ArgoCD lives on. Having those requests authenticated Allow ArgoCD to fetch the secret and inject them to the `Secret` objects without having the secrets stored in the Git repository.

---

### Creating required service account, clusterRole and clusterRoleBinding required for argoCD-vault integration
In order to have sent request from ArgoCD to vault authenticated, We need to adjust some of the configuration in ArgoCD and kubernetes side. Following set of action should be taken in ArgoCD Kubernetes cluster:
 - Create a Service Account and make a Secret Holding its Token
 - Create a `clusterRoleBinding` to bind the service account to the Kubernetes `system:auth-delegator`. This is necessary and allow all entities presenting the service account Token communicate with Kubernetes authentication mechanism. Vault requires to talk to Kubernetes and validate the tokens in requests coming from Kubernetes. Vault uses the token for the created `serviceAccount` and if the binding is not in place, Kubernetes will reject validating tokens vault is asking about.
  
  These two steps are crucial. We will provide vault with `serviceAccount` JWT token stored in the created service, along with Kubernetes CA to make vault request authenticated in ArgoCD cluster Kubernetes side.

  As ArgoCD is deployed via Helm, we can simply run the following command to have `ServiceAccount`, `clusterRoleBinding` and `AVP`(ArgoCD vault plugin) installed. This is important to note among 4 options we have to install a plugin on  ArgoCD, We will move on with `Sidecar` approach. We will have two sidecars in `argocd-repo-server` pod, responsible for addressing pure yaml files and helm.
  To have plugin installed along with configurations of service account, use the following
  ```
  kubectx kind-lab-argocd
  helm upgrade argocd argo/argo-cd -n argocd -f ./argocd-values.yaml
  ```
**Key Points**
 - AVP plugin is responsible to talk to Vault. As it resides inside `argocd-repo-server` deployment and uses `argocd-repo-server` serviceAccount, the steps for creating the secret and adding clusterRoleBinding will be applied for this SA
 - New secret `argocd-repo-server-secret` holding service account token will be created. You can get the JWT token using the following 
  ```
  kubectl -n argocd get secret argocd-repo-server-secret -o jsonpath="{.data.token}" | base64 -d
  ```
  - `argocd-repo-server-xxxx` pod will come up with 3 containers (repo-server,avp,avp-helm).
  
### Configuring Vault Kubernetes Authentication Method
We will add required configuration via Vault UI. Follow the menu path `Access > Enable new method > Kubernetes` . Set the path to `argocd-cluster`. This path is also used in AVP configuration and provided by env var `ARGOCD_ENV_AVP_K8S_MOUNT_PATH`. By enabling the method, its time to set configurations for this method.
 - Kubernetes host: The address in which Argocd Kubernetes cluster API is accessible with. If you took the same steps mentioned in the document, it should be `http://<your interface IP>:6443`
 - Kubernetes CA certificate: The CA certificate of ArgoCD kubernetes cluster. You can get it via following command
 ```
 kubectx  kind-lab-argocd
 kubectl exec argocd-repo-server-xxx-yyy -- cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
 ```
 - Token Reviewer JWT: This is the token for a service account bound with `system:auth-delegator`. In our case, As the AVP containers are using `argocd-repo-server` serviceAccount, this is the token for this service account, which can be retrived from secret created out of it.
  ```
  kubectl -n argocd get secret argocd-repo-server-secret -o jsonpath="{.data.token}" | base64 -d
  ```
Save the config and Kubernetes auth method will be created.
We need to define a role for this authentication method as well. All entities authenticated with this method, will get this role. By attaching this role to the previously created `policy`, authenticated entities now can also access the secret we created under `development` path.