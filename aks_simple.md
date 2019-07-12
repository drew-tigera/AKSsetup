# Azure TSEE Workshop - Azure Prep & AKS Cluster Deployment

## Prerequisites

You will need the following items to complete the labs.

  * General knowledge of Kubernetes and AKS
  * az CLI installed on your laptop
  * kubectl installed on your laptop
  * OpenSSH public and private RSA keys to add to your AKS cluster
  * Linux subsystem for Windows or a Linux VM could also be handy
  * [Integrate Azure Active Directory with Azure Kubernetes Service](IntegrateAzureActiveDirectory.md)

## Setup Your AKS Cluster

Create a resource group for the AKS cluster.

```
az group create --name myResourceGroup --location westus2
```

Create new VNET
```
az network vnet create \
  --resource-group myResourceGroup \
  --name myAKSVnet \
  --address-prefixes 10.0.0.0/8 \
  --subnet-name myAKSSubnet \
  --subnet-prefix 10.240.0.0/16
```

Associate Network resources with Node subnet

```
# Get the MC_ resource group for the AKS cluster resources
MC_RESOURCE_GROUP=$(az aks show --resource-group myResourceGroup --name myAKSCluster --query nodeResourceGroup -o tsv)

# Get the route table for the cluster
ROUTE_TABLE=$(az network route-table list -g ${MC_RESOURCE_GROUP} --query "[].id | [0]" -o tsv)

# Get the network security group
NODE_NSG=$(az network nsg list -g ${MC_RESOURCE_GROUP} --query "[].id | [0]" -o tsv)

# Update the subnet to associate the route table and network security group
az network vnet subnet update \
  --route-table $ROUTE_TABLE \
  --network-security-group $NODE_NSG \
  --ids $SUBNET_ID
```

Open up you command prompt of choice to run Azure CLI commands. Use az to build a small Kubernetes cluster.


Deploy the cluster.

```
az aks create --resource-group $RESOURCE_GROUP_NAME \
  --name $CLUSTER_NAME --generate-ssh-keys \
  --aad-server-app-id <YourServerAppID> \
  --aad-server-app-secret <YourServerAppSecret> \
  --aad-client-app-id <YourClientAppID> \
  --aad-tenant-id <YourTenantID> \
  --node-count 3 \
  --network-plugin azure \
  --service-cidr 10.0.0.0/16 \
  --dns-service-ip 10.0.0.10 \
  --docker-bridge-address 172.17.0.1/16 \
  --vnet-subnet-id $SUBNET_ID
```

Let this process run while the Tigera folks walk through the next section.

When your cluster is ready run the following command to configure kubectl to work with your new cluster

```
az aks get-credentials --resource-group <YOUR RESOURCE GROUP> --name <NAME FOR YOUR CLUSTER> --admin
```

Test your kubectl: (You should see 3 nodes running.)

```
kubectl get nodes
```
