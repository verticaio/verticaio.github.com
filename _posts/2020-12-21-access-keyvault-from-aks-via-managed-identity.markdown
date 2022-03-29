---
layout: post
title: Access Azure KeyVault from Azure AKS using Azure Managed Identity
date: 2020-12-21 06:30:20 +0300
description: Azure MI # Add post description (optional)
img: 2020-12-21-managed-identity.jpg # Add image post (optional)
fig-caption: Photo By Daniel Seßler on Unsplash # Add figcaption (optional)
tags: [Kubernetes, Azure, Identity, AzureDevOps]
---

So you are running your applications in Kubernetes but do you already have a solution for managing and storing all your application secrets? How do you tell Kubernetes to use the same source of truth for secrets avoiding secrets sprawl?  

# 1. Introduction

Applications need to handle user identities while also using their own to interact with other downstream services.  Azure Managed Identities (wrapper around service principal) for your applications enables you to securely connect to other Azure services without the need to manage and rotate secrets.

The image below describes the solution how an application running inside an Kubernetes Pod can access secrets in an Azure KeyVault with the help of Azure Managed Identities.

![design]({{site.baseurl}}/assets/img/2020-12-12-msi-design.png)

# 2. Deployment

A mandatory feature which needs to be enabled on our Kubernetes cluster is role based access control (rbac).  You can easily verify this executing the following Azure CLI command.  RBAC can only be enabled at creation time.

```
az resource show --resource-group <rg-aks-cluster> --name <aks-cluster-name> --resource-type Microsoft.ContainerService/ManagedClusters --query properties.enableRBAC
```
![rbac-enabled]({{site.baseurl}}/assets/img/2020-12-12-rbac-enabled.png)

## 2.1 Deploy AAD Pod Identity

**[AAD Pod Identity](https://github.com/Azure/aad-pod-identity)** enables Kubernetes applications to access cloud resources with Azure Active Directory.  Installing AAD Pod Identity is a straightforward process.
```
kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml
```
As a result, the Kubernetes API is extended with some custom resource defintions (CRD).
- **Managed Identity Controller (MIC)** is a central Pod with permissions to query the Kubernetes API server and checks for an Azure identity mapping that corresponds to a Pod.
- **Node-Managed Identity (NMI)** server listens for Pod requests to Azure services

After installing AAD Pod Identity, we verify the new API before proceding using *kubetctl api-versions*.
![apiversions]({{site.baseurl}}/assets/img/2020-12-12-apiversions.png)

The NMI has a daemonset definition which will deploy a Pod on each of the AKS nodes which are part of an virtual machine scaleset.
![daemonset]({{site.baseurl}}/assets/img/2020-12-12-daemon-nmi.png)

If a new Kubernetes Pod is brought online an identity will be assigned to it.  This action is visible in the logs of the Managed Identity Controller Pod.
![token-kv]({{site.baseurl}}/assets/img/2020-12-12-mic-log.png)

At the start we created our Kubernetes cluster with a paramater to also create an Azure managed identity.  In order to proceed with our soltion we need to extract some details about this identity. In order to do so, use the following command:

![identity]({{site.baseurl}}/assets/img/2020-12-12-identity-details.png)

## 2.2 Deploy CRD Azure Identity and binding

Next, we will define the Kubernetes Azure Identity and create the binding using the details of our Azure managed identity.

![podbinding]({{site.baseurl}}/assets/img/2020-12-12-podbinding.png)

1. Add the **name** of the Managed Identity
2. Fill in the **resourceID** of the Kubernetes MC (Managed Cluster) resourcegroup
3. The **clientId** of the Kubernetes Managed Identity
4. Define the **selector** value that will be used as label by your application Pods

After applying this Custom Resource Definition (CRD) to our cluster we can verify the new objects.
```
kubectl get azureIdentity 
kubectl get azureIdentityBinding
```
## 2.3 Configure RBAC for the AKS System-assigned managed identity

If you create an AKS cluster and you enable managed identity as authentication method, it will create the identity for your Azure virtual machine scale-set.
This system-assigned managed identity is behind the covers just an Azure Active Directory service principal which you can find back in your  **Azure Active Directory > Enterprise Applications**.

![azuread]({{site.baseurl}}/assets/img/2020-12-12-azuread.png)

We do have an identity now but without any role assignment.  The following commands assign different roles to the managed identity with specific scopes in order to fullfill the necessary requirements for our setup.

```
SUBID=$(az account show --query id -o tsv)
CLIENTID=$(az aks show --resource-group _rg-aks-p-we-001 --name aks-p-we-001 --query identityProfile.kubeletidentity.clientId -o tsv)
NODE_RESOURCE_GROUP=$(az aks show --resource-group _rg-aks-p-we-001 --name aks-p-we-001 --query nodeResourceGroup -o tsv)
MANAGED_IDENTITY=$(az aks show --resource-group _rg-aks-p-we-001 --name aks-p-we-001 --query identityProfile.kubeletidentity.resourceId -o tsv)

az role assignment create --role "Virtual Machine Contributor" --assignee $CLIENTID --scope /subscriptions/$SUBID/resourcegroups/$NODE_RESOURCE_GROUP
az role assignment create --role "Managed Identity Operator" --assignee $CLIENTID --scope /subscriptions/$SUBID/resourcegroups/$NODE_RESOURCE_GROUP
az role assignment create --role "Managed Identity Operator" --assignee $CLIENTID --scope $MANAGED_IDENTITY

AKS_RESOURCE_GROUP=$(az aks show --resource-group _rg-aks-p-we-001 --name aks-p-we-001 --query resourceGroup -o tsv)

az role assignment create --role "Virtual Machine Contributor" --assignee $CLIENTID --scope /subscriptions/$SUBID/resourcegroups/$AKS_RESOURCE_GROUP
az role assignment create --role "Managed Identity Operator" --assignee $CLIENTID --scope /subscriptions/$SUBID/resourcegroups/$AKS_RESOURCE_GROUP
```

## 2.4 Assign Azure KeyVault access policies

The last step is to provide access to the Azure KeyVault.  By default, only the creator of an Azure KeyVault has full access to secrets, keys and certificates.  In our scenario, we need to provide the secret management permissions to our System-assigned managed identity.  This will enable the Kubernetes Pods to get the secrets from the Azure Key Vault.

Make sure you add the system assigned managed identity in the access policies of the Azure KeyVault.

![azuread]({{site.baseurl}}/assets/img/2020-12-12-keyvault.png)

## 2.5 Pipeline

We have covered a lot of configuration steps.  To standardize the deployment and avoid human error, an Azure DevOps pipeline is made available.  You can find the pipeline **aks-mi-kv-pipeline.yml** in the Github repo link below.

# 3. Demo application deployment

To verify our solution, we will deploy the Azure AKS helloworld helm chart.
```
helm repo add azure-samples https://azure-samples.github.io/helm-charts/
helm install azure-samples/aks-helloworld --generate-name
```

The trick here is that the Pods of our demo application need to be labelled and should match with the selector from our PodIdentityBinding.  As a result the pod is bound to the Managed Identity
```
kubectl label pod <pod-name> aadpodidbinding=azure-pod-identity-binding-selector
```

# 4. Get secret from inside POD

To get confirmation that our setup was successfull, we need to enter into the Pod of our demo application.
```
k exec -it <pod-name> bash
```
Install jq in the pod because we want to manipulate the output.
```
apt update 
apt install jq -y
```
With the following command, we can trigger a request for an identity access token.  This URL (169.254.169.254) is only accessible from within Azure. We are requesting an access token for the KeyVault service. Now that we have this access token we can use this to obtain a secret from our KeyVault.
```
token=`curl 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fvault.azure.net' -H Metadata:true -s |jq -r '.access_token'`
echo $token
```
![token]({{site.baseurl}}/assets/img/2020-12-12-token.png)

We are looking for secrets from our Azure Keyvault (kvpwe001) and more specifically the **password** secret.  The token from our previous step is used in the authorization header as a bearer token.
```
value=`curl https://kvpwe001.vault.azure.net//secrets/password?api-version=2016-10-01 -H "Authorization: Bearer ${token}" -s | jq -r ".value"`
echo $value
```
We don't need to pass any credentials thanks to Aad Pod Identity configuration. Our request (=Pod with correct Azure Identity label) came from within Azure and we have setup the permission which identity can access the KeyVault.
![token]({{site.baseurl}}/assets/img/2020-12-12-secret.png)

# 5. Troubleshooting

The following commands will help you in case you run into issues:
```
kubectl get AzureIdentity -A -o yaml
kubectl get AzureIdentityBinding -A -o yaml
kubectl get AzureAssignedIdentities -A -o yaml # this will show which Pod(s) has been assigned to the managed identity
kubectl get AzureAssignedIdentities -A -o jsonpath='{range .items[*]}{@.spec.pod}{"\t"}{@.metadata.namespace}{"\t"}{@.spec.azureIdentityRef.metadata.name}{"\n"}'
```

*The configuration we used in this post can be found on <https://github.com/dewolfs/aks-managedIdentity-kv>.*