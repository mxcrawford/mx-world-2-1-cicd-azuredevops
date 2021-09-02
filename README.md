# CI/CD with Mendix & Azure DevOps

This is a step-by-step guide on using Azure DevOps for CI/CD with the Mendix platform and the tools needed. 

## Tips & tricks

### Azure DevOps

* When creating your Azure DevOps project, it helps to remove any spaces and replace them with "-"
  e.g. Project: "Mendix World CICD" should rather be "mendix-world-cicd"

### Azure Cloud

* When creating the Kubernetes cluster and all Azure resources, try to make sure the region is the same across the board. This helps to avoid issues when dealing with the Key Vaults or other globally unique resource names
  e.g. "West Europe"

## Overview

### These steps focus on also deploying inside Azure Cloud but can be adapted to use in any Kubernetes. This guide covers the following Mendix deployment options that have varying degrees of complexity:

#### Very Complex 
1. Mendix on Kubernetes Bring Your Own Cloud (on Azure Cloud) [Link](#option-1-mendix-on-kubernetes-bring-your-own)
#### Complex
2. Mendix for Private Cloud - Standalone (Azure Cloud + Kubernetes) [Link](#option-2-mendix-for-private-cloud-standalone)
#### Simplest
3. Mendix Public Cloud [Link](#option-3-mendix-public-cloud)

## Option 1 Mendix on Kubernetes Bring Your Own

For this Bring Your Own Cloud example we are using:

* Kubernetes Service - Azure Cloud
* Azure Storage
* Azure Container Registry
* Azure SQL Server & Elastic Pool
* Azure Keyvaults (for master secrets and app-specific secrets)

This option for Mendix deployment is the most complex of the three examples here and it is recommended to use the Mendix for Private Cloud offering if don't absolutely need to go this way (https://docs.mendix.com/developerportal/deploy/private-cloud)

A big thank you and full credit to [Clyde de Waal](https://github.com/MXClyde) and others inside the Mendix team who have put together a great [guide](https://github.com/MXClyde/mendix-kubernetes-azure) on how to setup a Bring Your Own Kubernetes approach with Mendix & Azure Cloud. Also how to encorporate this into an Azure DevOps Pipeline.  Please follow the instructions instructions very carefully for the most part but also be aware that this is not an official productized offering so the guide is not kept up to date in real time and may differ at the time of using. See link to guide below:

[https://github.com/MXClyde/mendix-kubernetes-azure](https://github.com/MXClyde/mendix-kubernetes-azure)

## Option 2 Mendix for Private Cloud Standalone

This approach is less complicated than the Bring Your Own Kubernetes option mainly because of the great products and services as part of Mendix for Private Cloud built and offered by Mendix that make the process of Infrastructure Orchestration for Mendix apps much easier. This makes use of the Mendix for Private Cloud offering which is a paid for and licensed product.

### 1 Kubernetes Cluster

First we will assume that you have a Kubernetes cluster installed with an Ingress Controller. You can use AWS EKS, Openshift or Azure AKS for your cluster, for this example, we are going to be using the Azure Cloud Kubernetes Service AKS.  This will require access to a Service Principal in Azure Cloud and may require higher permissions if you are not the owner of the Tenant and Subscription.

If you are interested in doing this with a local Kubernetes cluster not hosted with a cloud provider please consider having a look at the FREE Mendix Academy course that takes you through the process: [https://academy.mendix.com/link/path/101/Mendix-for-Private-Cloud](https://academy.mendix.com/link/path/101/Mendix-for-Private-Cloud)

#### Set some variables to be used in the commands to create the AKS Cluster

| Variable         | Example (Do not use)                | Description                                                     |
| ---------------- | ----------------------------------- | --------------------------------------------------------------- |
| RESOURCE_GROUP   | rg-my-cluster                       |  Name of your resource group for all cluster-related resources  |
| REGION_NAME      | westeurope                          |  Region where the cluster will be and any resources related     |
| SUBNET_NAME      | my-subnet                           |  Name of the Subnet available to the Cluster                    |
| VNET_NAME        | my-vnet                             |  Name of the Vnet created by the cluster                        |
| PRINCIPAL_APP_ID | 9ce3aa11-49ec-4959-b5e-3587ddg55f05 |  Id of the Service Principal                                    |
| PRINCIPAL_SECRET | 823HKbnjkDTLbkKtEju~X7fGO-1.Ga07eT  |  Secret for the Service Principal                               |
| AKS_CLUSTER_NAME | my-cluster                          |  Name of your Cluster                                           |
| ACR_NAME         | my-registry                         |  Name of the registry for all cluster related items             |

Using the Azure portal Cloud shell or a shell terminal with the Azure CLI:

``` 
RESOURCE_GROUP=rg-my-cluster && \
REGION_NAME=westeurope && \
SUBNET_NAME=my-subnet && \
VNET_NAME=my-vnet && \
PRINCIPAL_APP_ID=9ce3aa11-49ec-4959-b5e-3587ddg55f05 && \
PRINCIPAL_SECRET=823HKbnjkDTLbkKtEju~X7fGO-1.Ga07eT && \
AKS_CLUSTER_NAME=my-cluster && \
ACR_NAME=my-registry
```

1. Create Resource Group
```
az group create \
    --name $RESOURCE_GROUP \
    --location $REGION_NAME
```
2. Create VNET (You can use different address and subnet prefixes)
```
az network vnet create \
    --resource-group $RESOURCE_GROUP \
    --location $REGION_NAME \
    --name $VNET_NAME \
    --address-prefixes 10.0.0.0/8 \
    --subnet-name $SUBNET_NAME \
    --subnet-prefix 10.240.0.0/16
```
3. Get some more variable values needed for cluster creation

```
SUBNET_ID=$(az network vnet subnet show \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET_NAME \
    --name $SUBNET_NAME \
    --query id -o tsv)
```

```
VERSION=$(az aks get-versions \
    --location $REGION_NAME \
    --query 'orchestrators[?!isPreview] | [-1].orchestratorVersion' \
    --output tsv)
```
4. Create the Cluster (This may take some time to full complete)

```
az aks create \
--resource-group $RESOURCE_GROUP \
--name $AKS_CLUSTER_NAME \
--vm-set-type VirtualMachineScaleSets \
--load-balancer-sku standard \
--location $REGION_NAME \
--kubernetes-version $VERSION \
--network-plugin azure \
--vnet-subnet-id $SUBNET_ID \
--service-cidr 10.2.0.0/24 \
--dns-service-ip 10.2.0.10 \
--docker-bridge-address 172.17.0.1/16 \
--generate-ssh-keys \
--service-principal $PRINCIPAL_APP_ID \
--client-secret $PRINCIPAL_SECRET
```

5. Once created you can load in the cluster credentials into your kubeconfig file using the Azure CLI command get-credentials

```
az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME
```

6. Create Registry & Update the cluster with the resource group and Registry

```
az acr create \
    --resource-group $RESOURCE_GROUP \
    --location $REGION_NAME \
    --name $ACR_NAME \
    --sku Standard
```
```
az acr repository list \
    --name $ACR_NAME \
    --output table
```
```
az aks update \
    --name $AKS_CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --attach-acr $ACR_NAME
```

7. Install the Ingress Controller inside a namespace that will make sense e.g. mxingress

```
kubectl create namespace mxingress
```

```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```
```
helm install nginx-ingress stable/nginx-ingress \
    --namespace mxingress \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
```

8. Once created, note the external IP

```
kubectl get service nginx-ingress-controller --namespace mxingress -w
```

### 2 Initial Setup

Now that your cluster is created, we need to run the initial setup for the Mendix for Private Cloud setup and basic resources such as Database and File Storage. In this example for the Database provider we will use [Postgres](https://www.postgresql.org/) and for File Storage we will use [Minio](https://min.io/)

To do this we first create a namespace for the storage (You can use your own name, take note of it)

```
kubectl create namespace privatecloud-storage
```

Once this is done, we can go into Azure DevOps and start creating our Initial Setup Release.

As a basis, we are going to import a release that I created beforehand for the initial setup: [release_mx4pc_initialsetup.json](release_mx4pc_initialsetup.json)

TODO:
- How to import existing release
- How to Create Azure Resource Manager connection

### 3 Create Cluster in Cloud Portal

TODO:
Portal Screenshots
GUI setup and SCreens

### 4 Custom Resource Definitions
TODO:
Explanation and link to CRDs
Github .yaml file

### 5 App Onboarding / Deployment

TODO:
Build pipeline using mxbuild.py
Import existing pipeline

## Option 3 Mendix Public Cloud

TODO:
Import existing pipeline