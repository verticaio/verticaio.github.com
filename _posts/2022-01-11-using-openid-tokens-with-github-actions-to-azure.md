---
layout: post
title: 'Using OpenID Connect (OIDC) tokens with GitHub Actions and Azure'
date: 2022-01-11 14:30:20 +0300
description: 'oidc' # Add post description (optional)
img: 2022-01-11-oidc.jpg # Add image post (optional)
fig-caption: Photo By Simon Berger on Pexels # Add figcaption (optional)
tags: [ devops, github, git, oidc, azure, openid ]
---

When you work in a large organization that deploys to a cloud provider you probably need to adhere to certain security (credential management) policies.
You need to store a credential/token as an encrypted secret in GitHub and present that secret to the cloud provider every time it runs.  
Now when you are adopting the **[ Azure Cloud Adoption Framework](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/#azure-landing-zone-conceptual-architecture)**, you will end up with multiple Azure landing zones.  Taking care of secret rotation becomes a hassle quite quickly.

# 1. Introduction

Luckily, at GitHub Universe, the new OpenID Connect (OIDC) feature was announced.  GitHub Actions can now authenticate with cloud providers using OpenID Connect, generating *ephemeral deploy tokens* and removing the need for complex secret management.  GitHub OpenID Connect short-lived tokens enable secure secret management with frequently rotating credentials.

GitHub Actions can now use an *Azure Active Directory Federated Identity* to authenticate and execute deployments against Azure, **without the need of secrets or keys**!  

In this blogpost, we are going through the process of setting up Federated Identity and using it in a GitHub repository to securely deploy to Azure.

![design]({{site.baseurl}}/assets/img/2022-01-11-design.png)

1. Create Azure AD Application Registration in Azure AD Tenant  
2. Create Azure AD Service Principal in Azure AD Tenant using the object ID of the Azure AD Appliction registration
3. Assign the Azure Contributor role permission to the Azure AD Service Principal on Azure Subscription scope  
4. Configure Federated Identity Credentials on the Azure AD Application Registration  
5. Create GitHub secrets in the repository settings  
6. Set GitHub workflow permissions and use the Azure/Login GitHub Action  

## 1.1 Workflow Details

GitHub Actions helps us automate, customize and execute software development workflows including CI/CD right from the GitHub repository.  OpenID Connect support between GitHub and Microsoft Azure helps us to embrace the vision to make it easier to get started, easier to maintain and more secure to deploy when using GitHub Actions.

For each deployment, the GitHub Actions workflow will request an auto-generated OpenID Connect token. This token has all the metadata needed to get a secure, verifiable identity for the workflow that’s trying to authenticate to Microsoft Azure.  The GitHub action Azure/Login is using this token and present it to their respective clouds.

The cloud provider validates the claims in the OpenID Connect token against the cloud role definition and provides a short-lived access token. GitHub Actions and steps within the same workflow job can use this access token to connect and deploy to the cloud.  

The token expires when the workflow job completes.

*What is OpenID Connect (OIDC)?*  

![logo]({{site.baseurl}}/assets/img/2022-01-11-openid-logo.png){: style="float: left"}

**Open ID Connect** is an authentication protocol that is based on OAuth 2.2.  This enables developers & IT Pro to connect their applications with clouds and systems using a *secure password-less* authentication.  We can completely rely on the authentication & authorization security mechanisms of the cloud provider to manage the right access between the deployment workflows and cloud resources.  
  
# 2. OpenID Connect Configuration

The OpenID Connect configuration takes place in Azure Active Directory, Azure and GitHub.  

We have created a PowerShell script (github-oidc-azure/setup-githubAzureOIDC.ps1) which will take care of all the necessary steps.  You can find the PowerShell script in the GitHub repository URL at the end of this blogpost.

## 2.1 Azure Active Directory Configuration

We start by creating an Azure AD Application Registration.  When you do this from the portal, automatically an Azure AD Service Principal is also created.  Open the Azure AD Application Registration and go the new menu **Federated credentials** in **Certificates & secrets**.

![fic]({{site.baseurl}}/assets/img/2022-01-11-az-fic.png)

In the image above, you can see we have configured 3 Federated Credentials:
- Credential to filter on pull request events
- Credential to filter on a specific branch
- Credential to filter on a specific GitHub Environment

When looking at the details of a Federated credential identity, we can see that the **subject identifier** contains the filter values; in this case on branch main.

![fic2]({{site.baseurl}}/assets/img/2022-01-11-az-fic2.png)

## 2.2 Azure IAM Configuration

The GitHub Workflow is targetting our Azure Subscription using the Azure/Login action. We assign the **Contributor** role to our Azure AD Service Principal so that we have full control over our Azure Subscription.

![iam]({{site.baseurl}}/assets/img/2022-01-11-az-iam.png)

## 2.3 GitHub Configuration

The last step in our configuration is executed in GitHub.  The Open ID Connect configuration requires 3 secrets:
- Azure Subscription ID
- Azure Tenant ID
- Azure AD Application Registration App ID

![gh-secrets]({{site.baseurl}}/assets/img/2022-01-11-gh-secrets.png)

These can be easily created using the GitHub CLI.

Our GitHub Workflow also requires a small modification.  The **Azure/Login** action remains the same but we add the *permissions* section.

![logs]({{site.baseurl}}/assets/img/2022-01-11-gh-workflow.png)

# 3. Results  

## 3.1 GitHub

The GitHub Action logs indicate **Using OIDC authentication** and the login is successful.

![logs]({{site.baseurl}}/assets/img/2022-01-11-gh-logs.png)

## 3.2 Azure
In the Azure Subscription Activity logs we can find back more details about the operation which was executed using OIDC.  

![logs]({{site.baseurl}}/assets/img/2022-01-11-az-logs.png)

# 4. Automation

Below you can view the output of the PowerShell script which automates all the necessary steps described in this blog.

![logs]({{site.baseurl}}/assets/img/2022-01-11-script.png)

*The configuration we used in this post can be found on <https://github.com/dewolfs/github-oidc-azure>.*