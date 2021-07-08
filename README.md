# CI/CD with Mendix & Azure DevOps

This is a step-by-step guide on using Azure DevOps for CI/CD with the Mendix platform and the tools needed. 

## Getting started with Kubernetes, Azure DevOps & Mendix

### Tips & tricks

#### Azure DevOps

* When creating your Azure DevOps project, it helps to remove any spaces and replace them with "-"
  e.g. Project: "Mendix World CICD" should rather be "mendix-world-cicd"

#### Azure Cloud

* When creating the K8s cluster and all Azure resources, try to make sure the region is the same across the board. This helps to avoid issues when dealing with the Key Vaults or other globally unique resource names
  e.g. "West Europe"

### These steps focus on also deploying inside Azure Cloud but can be adapted to use in any Kubernetes. This guide covers the following Mendix deployment options:

* Mendix on K8s Bring Your Own Cloud (on Azure Cloud) [Link](#mendix-on-k8s-byoc)
* Mendix for Private Cloud - Standalone (Azure Cloud + k8s) [Link](#mendix-for-private-cloud-standalone)
* Mendix Public Cloud [Link](#mendix-public-cloud)

## Mendix on k8s BYOC

For this Bring Your Own Cloud example we are using:

* K8s Service - Azure Cloud
* Azure Storage
* Azure Container Registry
* Azure SQL Server & Elastic Pool
* Azure Keyvaults (for master secrets and app-specific secrets)

[https://github.com/MXClyde/mendix-kubernetes-azure](https://github.com/MXClyde/mendix-kubernetes-azure)

## Mendix for Private Cloud Standalone

## Mendix Public Cloud