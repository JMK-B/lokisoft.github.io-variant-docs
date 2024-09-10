az group create --name com-lokisoft-rg --location uksouth
az group create --name dev-lokisoft-rg --location uksouth

az ad app create --display-name "VariantManagementApp" a1e79a40-a5c1-4be8-8a96-55bb5d309941



az ad sp create --id a1e79a40-a5c1-4be8-8a96-55bb5d309941

use objectid return from this

az role assignment create --assignee 484229cf-f20f-4589-ab8d-afab70710b3e  --role Contributor --scope "/subscriptions/5ac1f0a2-a9b1-4e3a-8b6a-01df2cee9a01/resourceGroups/com-lokisoft-rg"


az role assignment create --assignee 683a1d73-a768-455b-9c24-6b512bdb994e--role Contributor --scope "/subscriptions/5ac1f0a2-a9b1-4e3a-8b6a-01df2cee9a01/resourceGroups/dev-lokisoft-rg"

![](Pasted%20image%2020240502131345.png)
az role assignment create --assignee <Service-Principal-Id> --role Contributor --scope <Storage-Account-ResourceId>


Once done ensure the following resourceproviders are registered : subscription->ResourceProviders
* Micrsooft.Storage
* Microsoft.Web
* ManageIdentity
* keyVaults
* Microsoft.Insights

az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.KeyVault
az provider register --namespace Microsoft.ManagedIdentity
az provider register --namespace Microsoft.Insights
  


=============================

az ad app list --display-name "PizzaExManagementApp"
