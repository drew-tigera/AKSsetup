
# Installing Azure AKS with Azure Active Directory Integration


### 1. Environment Vars

```
aksname="YourClusterName"
location="westus2"
```

### 2. Azure Service Principal creation and AD Setup


Please ensure that you are using the latest az cli version.

```
serverApplicationId="$(az ad app create --display-name "${aksname}Server" --identifier-uris "https://${aksname}Server" --query appId -o tsv)"

az ad app update --id $serverApplicationId --set groupMembershipClaims=All
az ad sp create --id $serverApplicationId
serverApplicationSecret=$(az ad sp credential reset --name $serverApplicationId --credential-description "AKSPassword" --query password -o tsv)
az ad app permission add --id $serverApplicationId --api 00000003-0000-0000-c000-000000000000 --api-permissions e1fe6dd8-ba31-4d61-89e7-88639da4683d=Scope 06da0dbc-49e2-44d2-8312-53f166ab848a=Scope 7ab1d382-f21e-4acd-a863-ba3e13f7da61=Role
az ad app permission grant --id $serverApplicationId --api 00000003-0000-0000-c000-000000000000 
az ad app permission admin-consent --id  $serverApplicationId

oAuthPermissionId="$(az ad app show --id $serverApplicationId --query "oauth2Permissions[0].id" -o tsv)"
clientApplicationId="$(az ad app create --display-name "${aksname}Client" --native-app --reply-urls "https://${aksname}Client" --query appId -o tsv)"

az ad sp create --id $clientApplicationId 
az ad app permission add --id $clientApplicationId --api $serverApplicationId --api-permissions $oAuthPermissionId=Scope
az ad app permission grant --id $clientApplicationId --api $serverApplicationId


az group create -n ${aksname} -l ${location} 
```

Create Service Principal

```
SP=$(az ad sp create-for-rbac -n "${aksname}SP" --skip-assignment)
```

Set Environment variables.
```
appId=$(echo $SP | jq -r .appId)
clientSecret=$(echo $SP | jq -r .password)

account="$(az account show --query '{subscriptionId:id,tenantId:tenantId,user:user.name}')"
tenantId="$(echo $account|jq -r .tenantId)"
subscriptionId="$(echo $account|jq -r .subscriptionId)"
```

### 3. Deploy AKS Cluster

Deploy AKS cluster without --network-policy option


```
az aks create \
  --resource-group   $aksname  \
  --name $aksname \
  --node-count 2 \
  --generate-ssh-keys \
  --aad-server-app-id $serverApplicationId \
  --aad-server-app-secret $serverApplicationSecret \
  --aad-client-app-id $clientApplicationId \
  --aad-tenant-id $tenantId \
  --service-principal $appId \
  --client-secret $clientSecret
```

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
