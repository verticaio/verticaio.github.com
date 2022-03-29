---
layout: post
title: Get productive with Azure Bicep using GitHub Actions
date: 2021-01-11 06:30:20 +0300
description: Get productive with Azure Bicep using GitHub Actions # Add post description (optional)
img: 2021-01-11-bicep.jpg # Add image post (optional)
fig-caption: Photo By Zachary DeBottis on Pexels # Add figcaption (optional)
tags: [ Azure, GitHub, Bicep, iac ]
---
Managing modern infrastructure presents many different challenges. While the main operational aspects of infrastructure like durability, availability, scalability, security are very important, there’s also one aspect which should enable and support all the others - automation.

You know that you need multi-environmental architectures with consistency among all environments. Ideally, infrastructure-as-code and usage of provisioning tools will empower you to:
- Avoid human error
- Automate repeated tasks
- Add cloud security practices
- Allow agility and flexibility

# 1. Introduction

A few months back, Microsoft presented the Project Bicep, which is meant to be the Next Generation ARM Templates.
Bicep is a Domain Specific Language (DSL) for deploying Azure resources declaratively. It is an abstraction over ARM and ARM templates.

**DO NOT USE BICEP ON YOUR PRODUCTION UNTIL IT GOES TO ([V0.3](https://GitHub.com/Azure/bicep/projects)) (ETA 1/31/21)**.

The image below gives an overview of what we will implement today.

1. The **Main.bicep** file is making use of Bicep modules filling in the required parameters.
2. An ARM template (JSON-file) is generated using the **bicep build** command.
3. The ARM template is constructed based on the **Main.bicep** file.
4. The ARM template is deployed towards an Azure Subscription.

![design]({{site.baseurl}}/assets/img/2021-01-11-design.png)

# 2. Bicep Language

The language has a more human readable syntax that is easier to work with compared to ARM templates.
Applying the DRY (Don't repeat yourself)-methodology, we have created some Bicep modules stored in our GitHub repository.

A module is a pointer to another bicep file. It has a name property and a set of parameters that are exposed by the bicep file.
The type of the resource which is both the resource type and the api version combined into a single string which collectively represents what a resource can look like.

![image]({{site.baseurl}}/assets/img/2021-01-11-bicep.png){: style="float: left"}

**Bicep** is a new declarative language for describing and deploying your Azure resources.  It compiles into an ARM template.
You author Bicep code but you run 'bicep build' command to turn it into an ARM template.

Once it is an ARM template you can deploy it through CLI, pipeline,...
This is a new syntax of an existing runtime so the ARM template runtime is not changing we're just make it easier to author.

The **aks** module we have created below has some parameters defined.

![bicep-module]({{site.baseurl}}/assets/img/2021-01-11-bicep-module.png)

The **vnetSubnetId** can be referenced because we have defined the **subnets array output** as part of our vnet module.

![bicep-output]({{site.baseurl}}/assets/img/2021-01-11-bicep-output.png)

# 3. GitHub Actions

![github]({{site.baseurl}}/assets/img/2021-01-11-github-actions.png){: style="float: left"}

GitHub Actions gives teams CI capabilities, helping developers merge and deploy code many times in a single day. The silent power of GitHub Actions lies in its ability to programmatically define just about any workflow to mirror your team’s processes.

GitHub Actions gives us the power to use our repositories to speed up the delivery of our software and applications, all from one central point of truth.

Before we can start making use of GitHub Actions we need an Azure AD service principal and the necessary permissions. The following command will provide us with the required details.

{% highlight shell %}
# create Azure AD service principal
az ad sp create-for-rbac --name "GitHubaction" --role contributor --scopes /subscriptions/xxxx-xx-xx-xx-xxxx --sdk-auth
{% endhighlight %}

The output of the command above will give the service principal details in order to proceed:
![sp]({{site.baseurl}}/assets/img/2021-01-11-sp.png)

## 3.1 GitHub Secrets

We are going to use this output in GitHub to provide the necessary permissions to deploy resources in our Azure Subscription.
In GitHub, open the **Settings** menu and select **Secrets**. 

Provide a name for our credential and copy/paste the complete JSON output. 

![secret]({{site.baseurl}}/assets/img/2021-01-11-github-secret.png)

We are going to make use of GitHub Actions to setup a pipeline for CI/CD.  In GitHub this is called a workflow which contains steps.
To login to our Azure subscription we make a reference to the secret we created before called **AZURE_CREDENTIALS**.

Some GitHub actions require the SubscriptionID.  Hard-coding SubscriptionIDs is bad practice.  
The SubscriptionID can be retrieved from the **Azure Login** step and we save it as an environment variable that we can call in other steps.

![azure-cred]({{site.baseurl}}/assets/img/2021-01-11-action-login.png)

## 3.2 Environments

Defining and managing your development, test, staging, and production environments using infra-as-code tools is a common practice. 

![env]({{site.baseurl}}/assets/img/2021-01-11-github-env.png)

GitHub environment protection rules enable seperation of concerns between deployment and development to meet compliance and security requirements.

The required reviewers environment protection rule will automatically pause a job trying to deploy to the protected environment and notifies the reviewers.

Additionally we add a required reviewer before a deployment can take place to our Production environment.
![protect]({{site.baseurl}}/assets/img/2021-01-11-github-env-protect.png)

# 4. Execution

Our workflow file (.GitHub/workflows/main.yaml) contains the definition when the steps should be executed. 
During deployment time you can view the progress in the **Actions** menu.
![progress]({{site.baseurl}}/assets/img/2021-01-11-github-progress.png)

When our pipeline successfully completes the deployment to our development stage, the process is halted in a **Wait** status because a reviewer first need to approve the results before continuing.

![approval]({{site.baseurl}}/assets/img/2021-01-11-github-approval.png)

![approval2]({{site.baseurl}}/assets/img/2021-01-11-github-approval2.png)

When all steps are finished, you can view the **Environments** status at the startpage of your GitHub project.

![env2]({{site.baseurl}}/assets/img/2021-01-11-github-env2.png)

## 4.1 Tags

A GitHub Actions workflow status badge can be added to your README project file to give you a quick glance on latest build/deploy status.
![badge]({{site.baseurl}}/assets/img/2021-01-11-github-badge.png)

*The configuration we used in this post can be found on <https://GitHub.com/dewolfs/bicep-github-actions>.*