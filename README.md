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

To import and use the existing pipeline we will follow the instructions under Step C from the advanced Kubernetes guide here: https://github.com/MXClyde/mendix-kubernetes-azure#step-c-setting-up-build-and-release-pipelines-using-azure-devops

Under Step C, follow the instructions for: 
1. Azure Resource Manager Service Connection 
   This is needed to connect to your Azure Kubernetes Cluster and load the credentials

2. Importing the initial deployment pipelne using the file: [release_mx4pc_initialsetup.json](release_mx4pc_initialsetup.json)

Lastly, once imported, we will need to customise the above release as follows for the respective steps:

1. Install Helm

You should not need to customise this for some time until a newer version of helm is needed

2. Install Kubectl

Make sure the version you use for Kubectl is compatible with the version of Kubernetes running in your Azure cluster

3. AKS Get Credentials

First make sure to select the Azure Resource Manager Service Connection that you created beforehand

And lastly customise the script to your cluster details under the "Inline Script" field e.g.

```
az aks get-credentials \
    --resource-group rg-my-cluster \
    --name my-cluster
```

4. Helm Repo Update

This should remain the same unless the chart locations change

5. Install Database (Postgres)

Customise the Script to reflect how you would like the Postgress installation to be (Maybe you use another database provider etc). Make sure to use the same namespace as you created a the beginning of the Initial Setup e.g:

```
helm install postgres-shared stable/postgresql \
--namespace=privatecloud-storage \
--set postgresqlPassword=Password1\! \
--set image.tag=9.6 \
--set persistence.size=1Gi
```

6. Install Storage (Minio)

Customise the script to reflect how you need your storage setup with a different provider for example or different details. Make sure to use the same namespace as you created a the beginning of the Initial Setup e.g:

```
helm install minio-shared stable/minio \
--namespace=privatecloud-storage \
--set accessKey=minioadmin \
--set secretKey=Password1\! \
--set persistence.size=5Gi
```

7. Last step

Now we can run this release pipeline by clicking "Create Release".

### 3 Create Cluster in Cloud Portal

Next we need to create the cluster inside the Mendix portal to help us with the installation of the Mendix Operator.

Go to https://privatecloud.mendixcloud.com/ and register a new, STANDALONE cluster on AKS. Select the operating system where you are running your shell connected to the cluster (easiest is to run it usnig Azure's Cloud shell) More documentation [here](https://docs.mendix.com/developerportal/deploy/private-cloud-cluster#:~:text=3.1%20Creating%20a%20Cluster&text=Click%20Set%20up%20Mendix%20for,Click%20Register%20Cluster.)


![alt text](https://s3.eu-central-1.amazonaws.com/mendixdemo.com/Screenshots/NewNamespace.png "NewNamespace")

![alt text](https://s3.eu-central-1.amazonaws.com/mendixdemo.com/Screenshots/DownloadInstaller.png "Download Installer")

Run the installer using the given command 
```
./mxpc-cli installer -n my-cluster
```
Now follow the steps to setup your cluster from Step 4 here: https://docs.mendix.com/developerportal/deploy/private-cloud-cluster


### 3.1 Database

Enter the same access details from the Initial Setup

Host: [install name].namespace.svc.cluster.local e.g.

```
postgres-shared-postgresql.privatecloud-storage.svc.cluster.local
```

### 3.2 File Storage

Enter the same access details from the Initial Setup

Endpoint: http://[install name].namespace.svc.cluster.local:9000 e.g.
```
http://minio-shared.privatecloud-storage.svc.cluster.local:9000  
```

### 3.3 Ingress

Domain name: The base domain you plan to use so for app1.example.com you will use:
```example.com```

You will need to own the domain and point it to the external address of the Kubernetes Cluster

### 3.4 Registry

Pull & Push URLs:

```
[registry_name].azurecr.io

e.g. mycontainerregistry.azurecr.io
```

### 4 Custom Resource Definitions

Once the Cluster setup has been done, its a great idea to learn a bit about Custom Resources with Kubernetes: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/

The Mendix for Private Cloud offering has the Mendix Operator to make use of Mendix Custom Resources (CRs). You can read more here: https://docs.mendix.com/developerportal/deploy/private-cloud-operator

We have templatized the Mendix Custom Resources yaml example to automate things somewhat - it can be further expanded on for your needs, an example can be found in this repo. We also have an example pipeline that makes use of these templatized versions for you to provide variables to be replaced in the file inside Azure DevOps:
Accp: [standalone-app-template-accp.yaml](standalone-app-template-accp.yaml)
Prod: [standalone-app-template.yaml](standalone-app-template.yaml)

### 5 App Onboarding / Deployment

### 5.1 Build Pipeline

Before we can onboard (and deploy) our app we need to create a .mda Deployment package and upload it to an internet-accessible URL. In our case we use a public Azure Blob storage (We RECOMMEND ABSOLUTELY using something more secure)

In this case we have an example Build pipeline that can be imported into Azure DevOps ([How to import](https://akhilsharma.work/import-export-ci-pipeline-from-azure-devops/))

The JSON file can be found here: [build_mx4pc_mda_k8s-testapp.json](build_mx4pc_mda_k8s-testapp.json)

Adjust the following details in the respective tasks for the pipeline you have now imported:

### Download mxbuild.py

Here we download a mxbuild.py script to help us build the correct Mendix mda package

No changes needed

### Run mxbuild.py

Put the Mendix TeamServer Username and Password (Email Address & Password for home.mendix.com) inthe the "Arguments" field in this task

To make this easier you can add Mendix.Username and Mendix.Password variables to your pipeline to pass through in an automated way

### Upload files to Azure Storage container

Here you will need to select your Azure Resource Manager Connection from the dropdown (we created this earlier)

Next make sure your Location is correct and storage account and container lines up with the Azure Blob Storage you have created to be used for the mda packages. 

If you are not using Azure Blob storage you will replace this step with the upload method of your choice to the location of your choice

### Write MDA URL to file

No changes unless you are NOT using Azure Blob Storage

If you are not using Azure blob storage you can replace the templated url with the Internet accessible URL you are using:

Replace ```https://$(Storage.name).blob.core.windows.net/$(Storage.container)/D:/a/1/s/builds/project_$(Build.SourceVersion)_$(Build.BuildId).mda``` with ```https://your-mda-url.mda```

### 5.2 Release Pipeline

Lastly we need to use this mda package we've created and write the rest of the details for our app into the CR yaml file and deploy it.

For this we have an example release pipeline that you can import into Azure DevOps which you can find here:[release_mx4pc_deploy-k8stestapp.json](release_mx4pc_deploy-k8stestapp.json)

Once imported you will first need to select the Build pipeline we created before, as the input/incoming artifact for the release:

![alt text](https://s3.eu-central-1.amazonaws.com/mendixdemo.com/Screenshots/BuildArtifact.png "ArtifactBuild")

Once the build pipeline is selected, you will need to customise some of the details for the tasks in the Release Pipeline:

### 1. Download App Template YAML

No changes needed unless you want to use a different template. This downloads the templatised CR files:

Accp: [standalone-app-template-accp.yaml](standalone-app-template-accp.yaml)
Prod: [standalone-app-template.yaml](standalone-app-template.yaml)

Use the correct file for the appropriate environment

Make sure all of the variables inside the template are available inside your release and are created when you run the release.  

![alt text](https://s3.eu-central-1.amazonaws.com/mendixdemo.com/Screenshots/Variables.png "Variables")

### 2. Replace tokens in **/*.yaml

No changes needed. This will take all of the tokens in the templatised .yaml file and replace them with the variable and system values

### 3. Deploy app to Kubernetes cluster

Here you will need to add a "Kubernetes service connection" to your Azure DevOps project similar to the "Azure Resource Management Connection" we added earlier.  For the non-Azure Kubernetes clusters, you can use the kubeconfig file to configure (More [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml#kubernetes-service-connection)) 

This will use the ```kubectl apply``` operation on the cluster using the .yaml CR file.

Now we should be able to run the release.

You can use 

```kubectl get mendixapp -n=my-namespace```
or
```kubectl get pods -n=my-namespace```

to check if this has worked as expected

## Option 3 Mendix Public Cloud

This is a nice and simple option and we have created an example build and release pipeline you can use:

Build: [build_mxcloud.json](build_mxcloud.json)
Release: [release_mxcloud.json](release_mxcloud.json)

These make use of the Build and Deploy APIs from the Mendix portal: https://docs.mendix.com/apidocs-mxsdk/apidocs/

You will need to update the service endpoints for each REST call or use a Web Service Endpoint: https://docs.microsoft.com/en-us/azure/devops/extend/develop/service-endpoints?view=azure-devops

This does require a Mendix Cloud license.
