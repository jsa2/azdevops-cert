## CLI Script (Cloud Shell, BASH)
Ensure you have set the default devops organization, and have updated the AZ CLI ``AZ CLI upgrade``
```bash
## Secure Azure Devops Service Connection creation 

## Creates Contributor in the resource subscription 

## Runs on AZCLI, but depends on depedencies that are pre-installed often with linux. Also line-break is [\] whereas in Powershell [`] 

## Set subscription context 
## Do fresh sign-in to ensure az devops commands will work
az login 
subName="Microsoft Azure Sponsorship"
az account set --subscription "$subName"
DevopsProject=TestProj
DevopsOrg=thx138
RoleOfSPN=Contributor
# Az devops default organization needs to be configured sucesfully before continuing running this script
az devops configure --defaults organization=https://dev.azure.com/$DevopsOrg

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

## Sign in to AzDevops and configure the org you want the ServiceConnection to be created as default

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
