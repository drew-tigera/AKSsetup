Please note values in angle brackets <YourValueHere> are meant to be replaced with your specific values. We use environment variables to facilitate this.
  
  
# Installing Azure AKS with Azure Active Directory and Azure CNI Integration


### 1. Environment Vars 

```
aksname="<YourClusterName>"
location="<YourAzureRegion>"
```

### 2. AD Setup 

Create Azure AD server component

```
serverApplicationId=$(az ad app create \
    --display-name "${aksname}Server" \
    --identifier-uris "https://${aksname}Server" \
    --query appId -o tsv)

```

Update the application group memebership claims

```
az ad app update --id $serverApplicationId --set groupMembershipClaims=All
```

Create Service Principal

```
az ad sp create --id $serverApplicationId
```
The output should look something like this:

```
{
  "accountEnabled": "True",
  "addIns": [],
  "alternativeNames": [],
  "appDisplayName": "aksclustername",
  "appId": "XXXXXXXX-8dd5-XXXX-XXXX-7f0e4fdfc3d6",
  "appOwnerTenantId": "b0497432-XXXX-4177-be42-XXXXXXXXXXX",
  "appRoleAssignmentRequired": false,
  "appRoles": [],
  "applicationTemplateId": null,
  "deletionTimestamp": null,
  "displayName": "aksclustername",
  "errorUrl": null,
  "homepage": null,
  "informationalUrls": {
    "marketing": null,
    "privacy": null,
    "support": null,
    "termsOfService": null
  },
  "keyCredentials": [],
  "logoutUrl": null,
  "notificationEmailAddresses": [],
  "oauth2Permissions": [
    {
      "adminConsentDescription": "Allow the application to access americanairServer on behalf of the signed-in user.",
      "adminConsentDisplayName": "Access aksclustername",
      "id": "XXXXXXXX-bec8-XXXX-9bb6-XXXXXXXXXXXXX",
      "isEnabled": true,
      "type": "User",
      "userConsentDescription": "Allow the application to access americanairServer on your behalf.",
      "userConsentDisplayName": "Access aksclustername",
      "value": "user_impersonation"
    }
  ],
  "objectId": "XXXXXX-c478-XXXX-9074-XXXXXXXXXXX",
  "objectType": "ServicePrincipal",
  "odata.metadata": "https://graph.windows.net/b0497432-28dd-4177-be42-7d4072ca6006/$metadata#directoryObjects/@Element",
  "odata.type": "Microsoft.DirectoryServices.ServicePrincipal",
  "passwordCredentials": [],
  "preferredSingleSignOnMode": null,
  "preferredTokenSigningKeyEndDateTime": null,
  "preferredTokenSigningKeyThumbprint": null,
  "publisherName": "Default Directory",
  "replyUrls": [],
  "samlMetadataUrl": null,
  "samlSingleSignOnSettings": null,
  "servicePrincipalNames": [
    "XXXXXXXXXXXXXXXXXXXX",
    "https://aksclustername"
  ],
  "servicePrincipalType": "Application",
  "signInAudience": "AzureADMyOrg",
  "tags": [],
  "tokenEncryptionKeyId": null
}
```
Get the service principal secret

```
serverApplicationSecret=$(az ad sp credential reset \
    --name $serverApplicationId \
    --credential-description "AKSPassword" \
    --query password -o tsv)
```

The Azure AD needs permissions to perform the following actions:

   Read directory data
   Sign in and read user profile
   
```
az ad app permission add \
    --id $serverApplicationId \
    --api 00000003-0000-0000-c000-000000000000 \
    --api-permissions e1fe6dd8-ba31-4d61-89e7-88639da4683d=Scope 06da0dbc-49e2-44d2-8312-53f166ab848a=Scope 7ab1d382-f21e-4acd-a863-ba3e13f7da61=Role
```
Then you need to grant these permissions to make them effective.

```
az ad app permission grant --id $serverApplicationId --api 00000003-0000-0000-c000-000000000000
```
The results should look like this:
```
{
  "clientId": "XXXXXXX-c478-XXXX-XXXX-605f334daa00",
  "consentType": "AllPrincipals",
  "expiryTime": "2020-08-01T15:33:09.556268",
  "objectId": "2mQ5fHjE8k6QdXXXXXXXXF3GG8eLXXXXXXXXLBtzM-g",
  "odata.metadata": "https://graph.windows.net/XXXXXXXX-28dd-XXXX--XXXX7d4072ca6006/$metadata#oauth2PermissionGrants/@Element",
  "odatatype": null,
  "principalId": null,
  "resourceId": "XXXXXXXX-b58b-XXXX-a138-73XXXXXXXXe8",
  "scope": "user_impersonation",
  "startTime": "2019-08-01T15:33:09.556268"
}
```
Then run this command:

```
az ad app permission admin-consent --id  $serverApplicationId
```

Next up create the Azure AD Client component.

```
clientApplicationId=$(az ad app create \
    --display-name "${aksname}Client" \
    --native-app \
    --reply-urls "https://${aksname}Client" \
    --query appId -o tsv)
```
Create a service principal for the client application

```
az ad sp create --id $clientApplicationId
```
The output should look like this:
```
{
  "accountEnabled": "True",
  "addIns": [],
  "alternativeNames": [],
  "appDisplayName": "myaskclustClient",
  "appId": "5c77a659-XXXX-XXXX-XXXX-2fcXXXXXXXX6",
  "appOwnerTenantId": "b0497432--XXXX-XXXX-XXXX-7d4XXXXXXXX6",
  "appRoleAssignmentRequired": false,
  "appRoles": [],
  "applicationTemplateId": null,
  "deletionTimestamp": null,
  "displayName": "myaksclustClient",
  "errorUrl": null,
  "homepage": null,
  "informationalUrls": {
    "marketing": null,
    "privacy": null,
    "support": null,
    "termsOfService": null
  },
  "keyCredentials": [],
  "logoutUrl": null,
  "notificationEmailAddresses": [],
  "oauth2Permissions": [
    {
      "adminConsentDescription": "Allow the application to access americanairClient on behalf of the signed-in user.",
      "adminConsentDisplayName": "Access myaksclustClient",
      "id": "0XXXXXd6-XXXX-40dd-XXXX-824XXXXXXXXa",
      "isEnabled": true,
      "type": "User",
      "userConsentDescription": "Allow the application to access americanairClient on your behalf.",
      "userConsentDisplayName": "Access myaksclusClitent",
      "value": "user_impersonation"
    }
  ],
  "objectId": "XXXXXXX2-XXXX-XXXX-XXXX-0b6XXXXXXXXc",
  "objectType": "ServicePrincipal",
  "odata.metadata": "https://graph.windows.net/bXXXXXX2-28dd-XXXX-XXXX-7dXXXXXXXX6/$metadata#directoryObjects/@Element",
  "odata.type": "Microsoft.DirectoryServices.ServicePrincipal",
  "passwordCredentials": [],
  "preferredSingleSignOnMode": null,
  "preferredTokenSigningKeyEndDateTime": null,
  "preferredTokenSigningKeyThumbprint": null,
  "publisherName": "Default Directory",
  "replyUrls": [
    "https://myaksclustClient"
  ],
  "samlMetadataUrl": null,
  "samlSingleSignOnSettings": null,
  "servicePrincipalNames": [
    "5c77a659-XXXX-4ea8-XXXX-2fcXXXXXXXX6"
  ],
  "servicePrincipalType": "Application",
  "signInAudience": "AzureADMyOrg",
  "tags": [],
  "tokenEncryptionKeyId": null
}
```

Get the oAuth2 ID for the server app to allow the authentication flow between the two app components

```
oAuthPermissionId=$(az ad app show --id $serverApplicationId --query "oauth2Permissions[0].id" -o tsv)
```

Add the permissions for the client application and server application components to use the oAuth2 communication flow 

```
az ad app permission add --id $clientApplicationId --api $serverApplicationId --api-permissions $oAuthPermissionId=Scope
```

Then run:

```
az ad app permission grant --id $clientApplicationId --api $serverApplicationId
```

The output should look like this:

```
{
  "clientId": "ee637172-XXXX-XXXX-XXXX-0bXXXXXXXX0c",
  "consentType": "AllPrincipals",
  "expiryTime": "2020-08-01T15:44:03.877688",
  "objectId": "cnFj7ofy-UOxWAtndf2FDXXXXXXXXXXXXXXXXXXXXgA",
  "odata.metadata": "https://graph.windows.net/b0497432-28dd-XXXX-XXXX-7dXXXXXXXX06/$metadata#oauth2PermissionGrants/@Element",
  "odatatype": null,
  "principalId": null,
  "resourceId": "7c3964da-c478-XXXX-XXXX-60XXXXXXXX00",
  "scope": "user_impersonation",
  "startTime": "2019-08-01T15:44:03.877688"
}
```

### 3. Deploy AKS Cluster

Create a resource group for the cluster:

```
az group create --name <YourResourceGroupName> --location $location
```
The output should look like this:

```
{
  "id": "/subscriptions/058e3078-XXXX-XXXX-XXXX-98aXXXXXXXX9/resourceGroups/myresourcegroup",
  "location": "centralus",
  "managedBy": null,
  "name": "myresourcegroup",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": null
}
```
Get your Azure tenant ID:

```
tenantId=$(az account show --query tenantId -o tsv)
```
Now build the actual cluster:

```
az aks create \
    --resource-group <yourResourceGroup> \
    --name $aksname \
    --node-count 3 \
    --generate-ssh-keys \
    --aad-server-app-id $serverApplicationId \
    --aad-server-app-secret $serverApplicationSecret \
    --aad-client-app-id $clientApplicationId \
    --aad-tenant-id $tenantId
    --network-plugin azure
```
This command will take a few minutes.


### Create New cluster-admin role

Create new cluster-admin user with admin context.

```
az aks get-credentials --resource-group  $aksname  --name $aksname --admin

# If the user is part of the respective AAD Tenant
userName="$(echo $account|jq -r .user)"

# Using objectId instead if the user is a guest user or not part of the AAD tenant
# userName="$(az ad signed-in-user show --query objectId -o tsv)"

cat <<-EOF|kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: contoso-cluster-admins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: ${userName}
EOF
```

Login as the new user and verify cluster access

```
az aks get-credentials --resource-group  $aksname  --name $aksname --overwrite-existing
kubectl get pods --all-namespaces
```
