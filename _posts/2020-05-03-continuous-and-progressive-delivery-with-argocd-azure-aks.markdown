---
layout: post
title: Continuous (GitOps) and progressive (canary) delivery with ArgoCD on Azure AKS
date: 2021-05-03 05:30:20 +0300
description: Continuous (GitOps) and progressive (canary) delivery with ArgoCD on Azure AKS # Add post description (optional)
img: 2021-05-03-argocd.jpg # Add image post (optional)
fig-caption: Photo By Lum3n Dow on Pexels # Add figcaption (optional)
tags: [ Azure, Kubernetes, AKS, ArgoCD ]
---

As the number of Kubernetes clusters under management increases, application owners and cluster operators need a programmatic way to approach cluster management.  

GitOps is a way to do continuous delivery by using Git as source of truth for declarative infrastructure and applications.  As changes are committed to the repo, linked clusters are automatically updated. Gitops keeps all your clusters consistent, version controlled, and reduces the administrative burden as you scale.  

<u>Benefits of GitOps:</u>

**1) Productivity** - Allows for simplied continuous delivery, which in turn lower the "Mean Time to Deployment".  
**2) Enhanced Experience** - Offer easy interface for user to act on the system. Hide the complexity of the system with tooling that everyone knows.  
**3) Stability** - Since all changes comes from one system, offer a higher stability from possible conflicts of multiple source of changes.  
**4) Reliability** - In case of problems, rollback is easy as revert commit in Git. Single source of truth simplifies the process and reduce delays.  
**5) Consistency** - One entry point for changes. Everthing is driven thru a commit in a Git repository.  Everything is described at one place. PR - Review - Merge - Apply  
**6) Security/Audit** - Leveraging Git build-in security and signature.  Allows for easy tracking of changes and their approvals
  

*Previously we already discussed [Adopting progressive (canary) delivery with Flagger and service mesh Linkerd on Azure AKS]({% post_url 2021-04-06-adopting-progressive-delivery-with-flagger-and-servicemesh-linkerd-on-azureaks %}) and [Automating Kubernetes using GitOps (reconciliation) with Azure DevOps Git]({% post_url 2020-10-14-automating-kubernetes-using-gitops-with-azure-devops-git %})* 


# 1. Introduction

**[Argo CD](https://argoproj.github.io/)** extends the benefits of declarative specifications and Git-based configuration management to accelerate deployment and lifecycle management of applications without compromising security and compliance.
![logo]({{site.baseurl}}/assets/img/2021-05-03-argologo.png){: style="float: left"}
ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. 

Application deployment and lifecycle management should be automated, auditable, and easy to understand. All this can be done using Argo.

Argo Rollouts is a Kubernetes controller and a set of CRDs which provide advanced deployment capabilities such as blue-green, canary, canary analysis, experimentation, and progressive delivery features to Kubernetes.

ArgoCD is composed of three mains components:

- **API Server:** Exposes the API for the WebUI / CLI / CICD Systems
- **Repository Server:** Internal service which maintains a local cache of the git repository holding the application manifests
- **Application Controller:** Kubernetes controller which controls and monitors applications continuously and compares that current live state with desired target state (specified in the repository). If a OutOfSync is detected, it will take corrective actions.

The image below gives a high-level overview:  

![gitclone]({{site.baseurl}}/assets/img/2021-05-03-architecture.png) 

# 2. Install Argo

Run the following commands to create the namespaces required for Argo and install the components:
{% highlight javascript %}
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://raw.githubusercontent.com/datawire/argo-rollouts/ambassador/release/manifests/install.yaml
{% endhighlight %}
## 2.1 Portal

Argo CD UI visualizes the entire application resource hierarchy, not just top-level resources defined in the Git repo. For example, developers can see ReplicaSets and Pods produced by the Deployment defined in Git. From the UI, you can quickly see Pod logs and the corresponding Kubernetes events. This turns Argo CD into very powerful multi-cluster dashboard.

{% highlight shell %}
kubectl port-forward svc/argocd-server -n argocd 8080:443
{% endhighlight %}
To login to the ArgoCD portal, we need to get the auto-generated password.  Execute the following command to retrieve it and access the portal with the username **admin**.
{% highlight shell %}
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
{% endhighlight %}
After you login, you will be presented with the following UI.

![argoportal]({{site.baseurl}}/assets/img/2021-05-03-argoportal.png)

## 2.2 Setup ArgoCD CLI

To interact with the API Server we need to setup the CLI:

**1) CLI login**  

When you have downloaded the **[ArgoCD CLI](https://argoproj.github.io/argo-cd/cli_installation/)**, we first have to authenticate to the ArgoCD API server.  Use the following command and login with the same username and password you have used for the portal.
{% highlight shell %}
argocd login localhost:8080
{% endhighlight %}
You should receive the message **'admin:login' logged in successfully**.

**2) Add Git repository**  

Next, we are going to add a private Git (Azure DevOps) repository which contains our Kubernetes manifests files.  In the top right corner of your Azure DevOps repository, click **Clone** and **Generate Git Credentials**.
![gitclone]({{site.baseurl}}/assets/img/2021-05-03-gitclone1.png)  
You will be presented with a username and password.  Use these 2 values in the command below:
{% highlight shell %}
argocd repo add https://<org_name>@dev.azure.com/<org_name>/<project_name>/_git/<repository_name> --type git --username username --password secret
{% endhighlight %}

**3) Create project**  

We can group multiple repositories & applications in a single project.  The following command will create a new ArgoCD project.  The GitOps mechanism works with a source (Git Repository) and destination (Kubernetes cluster) which should be kept declaratively in-sync.  The follwing command will create a new project:
{% highlight shell %}
argocd proj create kuard --dest https://kubernetes.default.svc,kuard --src https://<org_name>@dev.azure.com/<org_name>/<project_name>/_git/<repository_name> --allow-cluster-resource */* --description "kuard"
{% endhighlight %}

**4) Create application**  

The final step is to create an Application referring to our Git repository with exact path to our Kubernetes manifest files.
If a change is made in our Git repository we want to see this also in our Kubernetes cluster without human interaction (sync-policy=auto).  The following command will create a new application:
{% highlight shell %}
argocd app create kuard --repo https://<org_name>@dev.azure.com/<org_name>/<project_name>/_git/<repository_name> --path manifests --dest-namespace kuard --dest-server https://kubernetes.default.svc --auto-prune --project kuard --self-heal --sync-policy auto 
{% endhighlight %}

# 3. GitOps in Action!  

Now that we have done all the prep work, let's head over to the UI to verify our work.  

![app]({{site.baseurl}}/assets/img/2021-05-03-app.png)

Each application gets its own canvas with information about the project, status, repository, ...  The detailed application view gives you a lot of information.

![app]({{site.baseurl}}/assets/img/2021-05-03-argo-app.png)  
  
**1)** Application health  
**2)** Last repository sync status with tag, author and comment.  
**3)** Last sync result  
**4)** Visual representation of application hosted in Azure AKS.  

Our Kubernetes deployment manifest contains the container image and tag.
The Kubernetes service is of type LoadBalancer so we can acess our application externally via the Azure LoadBalancer.  

Currently we are running the **blue** version/tag of our application.

![app]({{site.baseurl}}/assets/img/2021-05-03-kuard-blue.png)

Lets now adjust our tag to **green** in the deployment Kubernetes manifest stored in our Git repository.  In a production environment, this process would be part of your automated CI workflow.
![app]({{site.baseurl}}/assets/img/2021-05-03-app-tag.png)

Instantly you will notice a change in the ArgoCD portal.  A sync is initiated and the old (=blue) Kubernetes Pods are deleted.
ArgoCD is monitoring our Git repository and has started new Kubernetes Pods with the new tag (=green).  

Also the last Git commit message is visible in the portal.  
![app]({{site.baseurl}}/assets/img/2021-05-03-argo-app-green.png)

Our application is now running the **green** version/tag of our application.  
![app]({{site.baseurl}}/assets/img/2021-05-03-kuard-green.png)

The **network** icon in the top right gives you also a nice view on the networking part of your application.  
![app]({{site.baseurl}}/assets/img/2021-05-03-network.png)

Additionally the **logs** of your Pods can be viewed from the portal.  
![app]({{site.baseurl}}/assets/img/2021-05-03-logs.png)

# 4. Progressive delivery

Progressive Delivery makes it easier to adopt Continuous Delivery, by deploying new versions to a subset of users and evaluating their correctness and performance before rolling them to the totality of the users, and rolled back if not matching some key metrics.  

Canary deployments is one of the techniques in Progressive Delivery, used in companies like Facebook to roll out new versions gradually. 

## 4.1 Argo Rollout

Argo Rollouts augments the functionality of the Kubernetes Deployment resource with additional deployment strategies.
A Rollout object is identical to a Deployment object except for a couple of keys fields(**apiVersion**, **Kind** and the **Strategy.**).

![app]({{site.baseurl}}/assets/img/2021-05-03-rollout.png)

Via the kubectl Argo plug-in, we can get more info about the rollout definitions in our Azure AKS cluster.

![app]({{site.baseurl}}/assets/img/2021-05-03-rollout2.png)

## 4.2 Canary release

When a developer makes a change to the application code, your CI-process will kick-in and execute automated tests.
The result of your CI workflow will be a new container image and tag stored in a container registry.

**1) Stage 0**  
We always start with a first deployment/rollout.  We keep watching the Argo Rollout object with the following command:
{% highlight shell %}
kubectl argo rollouts get rollout kuard --watch
{% endhighlight %}  
The the first revision (**stable**) is active.  
![app]({{site.baseurl}}/assets/img/2021-05-03-rollout-watch1.png)

**2) Stage 1**  
Via our GitOps configuration, ArgoCD has seen a change to our image tag in Git.  
To apply the new version(**canary**) of our application, the Argo Rollout canary strategy will be adopted as described in our manifest file.  50% of the traffic will be routed to our canary version of our application.  
![app]({{site.baseurl}}/assets/img/2021-05-03-rollout-watch2.png)

**3) Stage 2**  
Argo Rollout provides several ways to perform analysis to drive progressive delivery.
For now, we will promote the rollout manually via the CLI:
{% highlight shell %}
kubectl argo rollouts promote kuard -n kuard
{% endhighlight %}
The previously green (**canary**) version of our application has become the **stable** release.  
![app]({{site.baseurl}}/assets/img/2021-05-03-rollout-watch3.png)

*The configuration we used in this post can be found on <https://GitHub.com/dewolfs/argocd-aks>.*