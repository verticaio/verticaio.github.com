---
layout: post
title: Automating Kubernetes using GitOps (reconciliation) with Azure DevOps Git
date: 2020-10-14 14:30:20 +0300
description: GitOps # Add post description (optional)
img: 2020-10-14-automate-kubernetes-gitops.jpg # Add image post (optional)
fig-caption: Photo By Liger Pham on Pexels # Add figcaption (optional)
tags: [Kubernetes, Azure, GitOps, Flux, AzureDevOps]
---

15 years ago, Git changed the way software teams collaborate and develop software. For new declarative software systems such as Kubernetes, Git can play a key role in deploying, configuring, updating and managing infrastructure as code.  

Nowadays, Kubernetes is taking up an important role in delivering cloud native solutions, simplifying and accelerating deployments and allowing a much higher rate of updates and upgrades.

# 1. Introduction

GitOps relies on Git as a single source of truth for declarative infrastructure and applications. With Git at the center of delivery pipelines, developers can make pull requests to accelerate and simplify application deployments and operations tasks to Kubernetes.  GitOps is a way to do continuous delivery by using Git as source of truth for declarative infrastructure and applications. 

![image]({{site.baseurl}}/assets/img/2020-10-14-flux-logo.png){: style="float: left"}

**[Weave Flux](https://github.com/fluxcd/flux)** is an open source operator that makes GitOps happen in your Kubernetes cluster.  Flux joined the CNCF in August 2019 as a sandbox project.  
We will use Flux to quickly propagate changes to clusters.  It automatically ensures that the state of your Kubernetes cluster matches the configuration you’ve supplied in Git.  During reconciliation, the target Kubernetes resources match with the resources stored in Git and decide which resources should be updated/deleted/created.   

The first **[CNCF Technology Radar](https://www.cncf.io/blog/2020/06/12/introducing-the-cncf-technology-radar/)** on Continuous Delivery placed Flux in Adopt, indicating significant level of adoption and success with the technology.
![tech-radar]({{site.baseurl}}/assets/img/2020-10-14-tech-radar.png)

The three core components of GitOps are:  

**1. Git repository**: storage for your Kubernetes manifests  
**2. Agent**: a tool to get your manifests into the Kubernuetes cluster  
**3. Kubernetes**: a Kubernetes cluster hosting your application defined in the Git repo  

Why do we want to do GitOps?
- Single source of truth
- Security, compliance and auditing
- Developer centric
- Trivialises rollback
- Declarative
- Observable - detect configuration drift
- Velocity

The image below gives you an overview how GitOps acts between the different components.

![design]({{site.baseurl}}/assets/img/2020-10-14-gitops-design.png)

We will explore the GitOps methodology and see the benefits of using Flux to do Kubernetes cluster management and application delivery. 

# 2. Deploy Weave Flux GitOps Kubernetes operator

As a first step we need to deploy the Flux operator into our Azure Kubernetes cluster.  At the root of our repository you can find the Azure DevOps pipeline **install-flux-pipeline.yml** to automate the installation and configuration of Flux.

![install]({{site.baseurl}}/assets/img/2020-10-14-flux-install.png)

The Helm **values.yaml** file contains the Helm settings for our Flux chart applied at install time.  The **git.url** value can be found when you clone an Azure DevOps repository under SSH.

![ssh]({{site.baseurl}}/assets/img/2020-10-14-devops-ssh.png)

## 2.1 Verify Flux deployment

By verifying the logs of our Flux deployment we can be sure our operator is in a good state.  You can use the following command:

```
kubectl -n flux logs deployment/flux -f
```

When we use the *describe* command on our Flux pod, we can confirm the applied settings as defined before in our Helm **values.yaml** file.

```
kubectl describe pod -n flux <insert flux pod name>
```

# 3. Setup connection between Flux and Azure DevOps

When Flux is up & running inside our Azure Kubernetes cluster, it automatically generates a SSH key-pair.  We need to add the public SSH key (existance inside the flux container) to Azure DevOps to allow authentication between Flux and our Azure DevOps code repository.

When the Flux pod starts, it generates the SSH key and output the public key part to the log. To retrieve the public SSH key, use the following command:

```
kubectl -n flux logs deployment/flux | grep identity.pub | cut -d '"' -f2
```
Copy the public SSH key and open your **user properties** in Azure DevOps > **SSH Public Keys**.  Give the SSH key a name and paste the SSH key in the Public Key Data.

![keys]({{site.baseurl}}/assets/img/2020-10-14-devops-ssh-keys.png)

## 3.1 Troubleshooting Flux

If you encounter connectivity issues when looking into the Flux deployment log file, try to do the Git clone operation manually.  Enter the Flux container by using the following command:

```
kubectl exec -it -n flux flux-7b588f586b-mrcbw -- /bin/bash
```

Clone your Azure DevOps repository from inside the Flux container with the following command:

```
git clone git@ssh.dev.azure.com:v3/<azure_devops_org_name>/<project_name>/<repo_name>
```

# 4. GitOps in action
If you check back the Flux deployment log, you will notice movement and if you dissect the log, some familiar Kubernetes commands start to light up.

![log]({{site.baseurl}}/assets/img/2020-10-14-flux-log.png)

The pods of our Azure Vote application have been created via the Flux operator.

![pods]({{site.baseurl}}/assets/img/2020-10-14-pods.png)

We can now access our Azure Vote application via the Kubernetes external service IP.

![catsdogs]({{site.baseurl}}/assets/img/2020-10-14-cats&dogs.png)

## 4.1 Reconsilation

This was the first step in the right direction but in real life our application will be updated with new features on a regular basis.  A Continuous Integration workflow is necessary to test and build our application.  The next step is to create a container image and push it inside a container registry with a release tag.  

To simulate this, we change the container image tag in our Kubernetes YAML file to image *mcr.microsoft.com/azuredocs/azure-vote-front:**v2***, the GitOps operator will notice a change in Git and act upon this change.  If we now access our Kubernetes external service IP again, we can see that the choices of our Voting app are now blue & purple and no more cats & dogs as before.

![bluepurple]({{site.baseurl}}/assets/img/2020-10-14-blue&purple.png)

In Azure DevOps, a tag has been created by the Weave Flux operator.  The Git commit SHA is also retrievable inside our Kubernetes deployment log file.

![tag]({{site.baseurl}}/assets/img/2020-10-14-tags.png)

*The configuration we used in this post can be found on <https://github.com/dewolfs/flux-gitops>.*