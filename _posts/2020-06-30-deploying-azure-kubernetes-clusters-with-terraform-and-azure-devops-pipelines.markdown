---
layout: post
title: Deploying Azure Kubernetes clusters with Terraform and Azure DevOps pipelines
date: 2020-06-30 08:30:20 +0300
description: Deploying Azure Kubernetes cluster with Terraform and Azure DevOps pipelines # Add post description (optional)
img: 2020-06-30-Terraform-and-DevOps.jpg # Add image post (optional)
fig-caption: Photo By Josh Sorenson on Pexels # Add figcaption (optional)
tags: [Azure, Terraform, DevOps, Kubernetes]
---
Cloud infrastructure automation is essential to accelerate the delivery of cloud-native applications by eliminating manual tasks.  The current wave of cloud computing is offering new strategies and a model for highly distributed applications.  It is an entirely new and different approach to building infrastructure, developing applications and structuring your teams.

Building on our experience with Terraform from our [previous post]({% post_url 2020-06-08-getting-started-with-terraform-on-azure %}), we will improve our workflow by adopting a CI/CD pipeline for our Terraform configuration to deploy multiple resources in Microsoft Azure.

# 1. Introduction

All the good design principles about CI/CD that we apply for applications are also applicable for Infrastructure-~~as~~-Is-Code.

The enterprise code-to-cloud challenges we encounter nowadays are:
- ensure repeatable and consistent deployment
- get end-to-end traceabililty
- perform safe deployments
- apply security and compliance at scale

With Azure DevOps pipelines we can build more agile practices and shift into a more rolling product-delivery approach for software and service development.

In this post, you will find more information how on to setup an Azure DevOps pipeline to deploy Azure Kubernetes clusters via Terraform.  The image below resembles what will create.

<p align="center">
<img src="{{site.baseurl}}/assets/img/2020-06-30-DevOps-Design.png">
</p>

# 2. Code Repository structure

Our code repository contains multiple folders.

- **00_law:** The configuration for an Azure Log Analytics workspace with two solutions connected to our Azure Kubernetes cluster.
- **01_vnet:** We provision an Azure virtual network with multiple subnets; one reserved for our nodepool of the Azure Kubernetes cluster
- **02_acr:** We plan to use the private Azure Container Registry which will host our application Docker container images.
- **03_aks:** Creation of our Azure Kubernetes cluster in the predefined virtual network and linked to our precreated Azure Log Analytics workspace.
- **scripts:** This folder contains the scripts to provision the Azure resources to store the Terraform state files.
- **templates:** To optimize readability of our IaC Azure DevOps pipeline we have broken it down into smaller parts.

An important change in our Terraform configuration has been made where we have removed our service principal Id and secret together with the Tenant Id and Subscription Id.  These are still required to execute Terraform configuration but we define them now as an Azure DevOps service connection (link between Azure Devops and Azure Subscription). 

# 3. Terraform configuration

Our Terraform configuration file to create the Azure Log Analytics workspace does also contain a section to enable two solutions.

![terraform]({{site.baseurl}}/assets/img/2020-06-30-Terraform.png)

The **00_law/variables.tf** file has the solutions variable defined as a map.  The values for the two solutions that we want to install are defined in the **00_law/terraform.tfvars** file.  Finally we can loop through each value using the for_each argument in the **00_law/law.tf** file.

# 4. Link between Azure DevOps and Azure Subscription

The Azure service principal we have created before will be in charge to execute the deployment via Azure DevOps into our Azure Subscription.  Follow the steps below to create the service connection between Azure DevOps and our Azure Subscription.  The Azure Subscription connection name we will use later in our YAML pipeline.

|**Step 1**|**Step 2**|
|--|--|
|![service-conn]({{site.baseurl}}/assets/img/2020-06-30-Service-connection.png) | ![service-conn]({{site.baseurl}}/assets/img/2020-06-30-Service-connection2.png) |

Take a look at my previous post if you are unsure how to get the Subscription ID, Tenant ID and the Service Principal Id and secret.

# 5. Setup Terraform backend

Next we are going to create the required resources to store the Terraform statefiles. We want to have complete environment isolation and therefor we create a development and a production storage account to store our statefiles.  The Azure CLI commands can be found in the Github repository "terraform-azuredevops > scripts > init-terraform-remote-state.sh".

Clone the repository (see link at the end of this post) and launch the shell script.

![cli-output]({{site.baseurl}}/assets/img/2020-06-30-cli-output.png)

# 6. Azure DevOps Pipeline

The main **azure-pipelines.yaml** file is calling several child-pipelines (ex. **templates/deploy-law.yaml**) for the different Azure resources we are provisioning. 

{% highlight shell %}
stages:
- stage: dev
  displayName: 'deploy to dev'
  jobs:
  - template: templates/deploy-law.yml
    parameters:
      serviceConnection: 'azure-conn'
      environmentName: 'dev'
      backendAzureRmStorageAccountName: '$(storageAccountNameDev)'
{% endhighlight %}

 The first job in the dev stage is calling the **templates/deploy-law.yml** (see below) pipeline to provision our Azure Log Analytics Workspace.  We are injecting the parameters at runtime and use these parameter references throughout our child-pipeline.

![yaml]({{site.baseurl}}/assets/img/2020-06-30-AzureDevOps-yaml.png)

This makes the main pipeline easier to read and we apply the DRY (Don't Repeat Yourself) principle by using the adequate values for the child-pipeline parameters depending on the stage we are in.

## 6.1 Pre-Deployment Approvals
Configuring approval between stages ensures that we get two pairs of eyes to ensure everything is deployed as expected before we continue to the next stage.

To specify who is required to approve the environment go to Azure DevOps > Environments > Approval and checks.

![pipeline]({{site.baseurl}}/assets/img/2020-06-30-AzureDevOps-approval.png)

A manual approval check on an environment ensures that deployment can happen only after the reviewers have signed-off.

![pipeline]({{site.baseurl}}/assets/img/2020-06-30-AzureDevOps-pipeline1.png)

Azure DevOps does also provide us with a complete log file of all the tasks which have been executed on the hosted agent.

![pipeline]({{site.baseurl}}/assets/img/2020-06-30-AzureDevOps-output.png)

*The configuration we used in this post can be found on <https://github.com/dewolfs/terraform-azuredevops>.*
