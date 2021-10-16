
# Azure Devops - Secure service connection with certificate
## Solution description
Creates Application and service principal with certificate credentials in Azure AD and deploys service connection to Azure Devops via AZ CLI
- ⚠ Removes the certificate files locally after deployment. Only Azure Devops has the private key after the deployment has completed.
- ✅ Azure AD itself does not know the private key(Azure AD only has the public key)
  
If you are looking to do this partially in GUI  - please see this guide [here](https://securecloud.blog/2021/04/13/azure-devops-use-certificate-for-azure-service-connection-spn/#guide)

- [Azure Devops - Secure service connection with certificate](#azure-devops---secure-service-connection-with-certificate)
  - [Solution description](#solution-description)
  - [Prerequisites](#prerequisites)
    - [Running in Cloud shell](#running-in-cloud-shell)
  - [CLI Script](#cli-script)
    - [Clean-up](#clean-up)
    - [Extra](#extra)
  - [Disclaimer](#disclaimer)


## Prerequisites 

Requirement | description | Install
-|-|-
✅ Bash shell script | Tested with ``WSL2/Ubuntu`` <br> ``wsl --list --verbose``   on windows | [CLI script](#cli-script)
✅ AZCLI | Azure CLI provides commands both for Azure AD and Azure Devops |``curl -sL https://aka.ms/InstallAzureCLIDeb \| sudo bash``
✅ AZCLI-devops extension|  Needed for Az devops commands |``az extension add --name azure-devops``

- Example is created for linux due to ease of setup ( Openssl is available out of the box in command line )
- If you want to run the example on windows declare variables in the script as windows variables "$varname=" and ensure you can refer to Openssl commands directly from cli
  - line-break is \ whereas in Powershell `

### Running in Cloud shell
Running in cloud shell seems not to support the existing session, or new session via Az Login, so you need to use PAT Token

--- 

access the cloud shell script [**here**](cloudShell.md) 

---

## CLI Script
Ensure you have set the default devops organization, and have updated the AZ CLI ``AZ CLI upgrade``
```bash
## Secure Azure Devops Service Connection creation 

## Creates Contributor in the resource subscription 

## Runs on AZCLI, but depends on depedencies that are pre-installed often with linux. Also line-break is [\] whereas in Powershell [`] 

## Set subscription context 
## Do fresh sign-in to ensure  "az devops commands now support sign-in through az login"
subName="Microsoft Azure Sponsorship"
az account set --subscription "$subName"
DevopsProject=TestProj
DevopsOrg=thx138
RoleOfSPN=Contributor
# Az devops default organization needs to be configured sucesfully before continuing running this script
az devops configure --defaults organization=https://dev.azure.com/$DevopsOrg

# Before continuing, ensure az devops project list command works (This will confirm, that you are logged in)
az devops project show -p $DevopsProject

## Generate certificate for 2 years
mkdir keys
openssl genrsa -out keys/private1.pem 2048
openssl req -new -x509 -key keys/private1.pem -out keys/public1.pem -days 720 -subj "/C=FI/CN=spnforaad.localdom/OU=IT Department/"
openssl pkcs12 -inkey keys/private1.pem -in keys/public1.pem -export -out keys/pack.pfx -passout "pass:mypass"
openssl pkcs12 -in keys/pack.pfx -passin "pass:mypass"  -out "keys/PemWithBagAttributes.pem" -nodes
# rm keys/ -r; mkdir keys

Sub=$(az account show -o tsv --query "id" )
Tid=$(az account show -o tsv --query "homeTenantId" )
## if you want below sub scope append the resource group, or resource to the scope
scope="/subscriptions/$Sub"
spnName="$DevopsOrg - CertConnection for sub - $Sub - $RoleOfSPN"

CLIENTCREDENTIALS=$(az ad app create --display-name "$spnName" \
-o tsv --query "appId")

az ad app credential reset --id $CLIENTCREDENTIALS --cert "@keys/public1.pem" --append
spn=$(az ad sp create --id $CLIENTCREDENTIALS -o tsv --query "objectId")

az role assignment create --assignee $spn \
--role $RoleOfSPN \
--scope $scope

endpoint=$(az devops service-endpoint azurerm create \
--azure-rm-service-principal-certificate-path "keys/PemWithBagAttributes.pem" \
--azure-rm-tenant-id $Tid  \
--azure-rm-subscription-id $Sub \
--azure-rm-service-principal-id $spn \
--name "$spnName" \
--azure-rm-subscription-name "$subName" \
--project "$DevopsProject" \
--output tsv --query "id")

echo $endpoint

## Delete the keys that were created
rm keys -r

az devops logout

## After deployment only Azure DevOps has the private key, which prevents misuse of the certificate credentials 
```

### Clean-up
Remove resources If you have same session open, which you deployed the solution
```bash

az devops service-endpoint delete \
--id "42460c3a-71c1-46df-b1dd-6802cb39016f" \
--project "$DevopsProject"

az role assignment delete --assignee $spn --role $RoleOfSPN
az ad app delete --id $CLIENTCREDENTIALS


```

### Extra
If you want to go the full nine yards on hardening the service connection, there is preview option to disable any password use on the service connection. This options is at the moment of writing this guide in preview [Application authentication method policies](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/whats-new#public-preview----application-authentication-method-policies)

## Disclaimer
The information in this document is provided “AS IS” with no warranties and confers no rights.