---
layout: post
title: Using policy-as-code to meet evolving company governance requirements on Azure AKS
date: 2021-06-02 05:30:20 +0300
description: kyverno # Add post description (optional)
img: 2021-06-02-kyverno.jpg # Add image post (optional)
fig-caption: Photo By Ben Mack on Pexels # Add figcaption (optional)
tags: [ Azure, Kubernetes, AKS, Kyverno, Policy, Bicep, k8s ]
---

Kubernetes is powerful, but configurations can be complex to configure and manage at scale. Policies allow separation of concerns and enforcement of security and best practices compliance. 

*In a previous post we touched on [enforcing and validating security posture with Open Policy Agent Gatekeeper on Azure AKS]({% post_url 2020-09-28-enforce-and-validate-security-posture-with-opa-gatekeeper-on-azure-aks %})*.

# 1. Introduction

![kyverno-logo]({{site.baseurl}}/assets/img/2021-06-02-kyverno-logo.png){: style="float: left"} **[Kyverno](https://kyverno.io/)**, a CNCF sandbox project is a policy management tool designed for Kubernetes. With Kyverno, cluster administrators can easily validate, mutate, and generate configurations _without_ the complexity and hassle of another language or external tools.

**Policy-as-Code** enables continuous compliance and protects against common misconfigurations. Kyverno enables Kubernetes native policy as code in a simple and scalable manner.

Some of the use-cases are:
- pod security
- additional security validation and enforcement
- fine-grained rbac
- multi-tenancy
- auto-labeling

In this blogpost, we will show how Kyverno works and demonstrate using Kyverno to address Kubernetes best practices and security across workloads to improve observability and security.  The image below gives an overview of a workflow controlled by Kyverno.

![design]({{site.baseurl}}/assets/img/2021-06-02-kyverno-design.png)

# 2. Installation

## 2.1 Infrastructure

Deploying and configuring your Azure services with Infrastructure-as-Code is an important step on any DevOps journey.  To spin up the required resources we are using Bicep which is a domain specific language.  The details can be found in **main.bicep** file.  At the root of our repository you can find the Azure DevOps pipeline **1-infra-aks-pipeline.yml** to automate the installation and configuration of the Azure AKS cluster.

## 2.2 Kyverno

In our last blogpost we discussed how to [Automate Kubernetes using GitOps (reconciliation) with Azure DevOps Git]({% post_url 2020-10-14-automating-kubernetes-using-gitops-with-azure-devops-git %}).  

GitOps is the technological core required for compliance automation, large-scale operations and lowering the cost of compliance & governance as well as lowering the cost of internal tooling.

We are going to review the basics of implementing policy controls using Kyverno and go in-depth on bringing those policy checks back into branch-commit-merge process.  With this setup we will be bringing policy enforcement to the left and identifying policy violations much earlier.

To make our lifes easier, we use a Makefile to automate the different steps in order to setup GitOps.

1. Setting the right Kubernetes cluster context  
2. Install ArgoCD
3. Setup GitOps to sync Kyverno manifest files  
4. Install Kyverno Reporting UI via Helm  
5. Install Kyverno security policies  
6. Install Kyverno custom policies

![makefile]({{site.baseurl}}/assets/img/2021-06-02-makefile.png)

Every part can be run seperately or all together via the command:
```
"make kubectx" or "Make all"
```
After GitOps done his magic, you should have a similar end-result in the ArgoCD portal.

![argo-final]({{site.baseurl}}/assets/img/2021-06-02-argo-final.png)

# 3. Policy configuration

Now that we have everything in-place, let's take a look at the policies.  To get an overview of the Kyverno policies, run the following command:

```
kubectl get clusterpolicy
```
The action column is important.  The values can either be audit or enforce.
- **audit:**report on matching resources which violate the rule(s)
- **enforce:** block insecure or non-compliant configurations

![cpol]({{site.baseurl}}/assets/img/2021-06-02-cpol.png)

If we take a closer look at the **require-run-as-non-root** policy, it is clear this policy applies to Pod resources inspecting the field **spec.containers[*].securityContext.capabilities** must be empty.
```
kubectl get cpol require-run-as-non-root -o yaml
```
![cpol]({{site.baseurl}}/assets/img/2021-06-02-cpol-spec.png)

# 4. Test-Cases

## 4.1 Bad pod
On Github, (<https://github.com/BishopFox>), you can find back examples of bad Kubernetes configurations which we can use to verify our policies.
![bishop]({{site.baseurl}}/assets/img/2021-06-02-bishopfox.png)

A picture is worth a thousand words so we apply the configuration and see what we get:
```
kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/everything-allowed/pod/everything-allowed-exec-pod.yaml
```

![bishop-block]({{site.baseurl}}/assets/img/2021-06-02-bishop-block.png)

The response above shows the name of the policy and the name of the rule which blocked the creation of the Pod, useful when backtracking the root-cause.

## 4.2 Mutate/Generate policies

In this section, we will apply the policies that are visible in the workflow at the top of this blogpost.
We are going to create a new Kubernetes namespace coupled with mutate and generate activities triggered by Kyverno.

- If the namespace contains the label compliance with value **iso27k** or **gdpr**, our company policy requires to add a new label **backup-enabled=true**.
- If the namespace contains the label networkzone with value **dmz**, our company policy demands to create a new Kubernetes network policy **deny all ingress/egress** traffic.
- If the namespace contains a not empty **departement** label, our company policy instructs to create a Kubernetes **resource quota cpu/memory** limit.
- If a new namespace gets created, our company policy requires to modify the name of the namespace.

The Kyverno policy definitions to setup the above configuration can be found in the repository folder **kyverno-policies**.

The **add_ns_quota.yaml** policy is using the **enforce** validation together with the **precondition check** to make sure the department label is not empty.  **Synchronize = true** will allows us to make changes to the policy in a later phase in GIT and the changes will be synchronized to the existing deployments.

![quota]({{site.baseurl}}/assets/img/2021-06-02-quota.png)

We create the following namespace:

```
apiVersion: v1
kind: Namespace
metadata:
  name: "thebank"
  labels:
    department: "sales"
    env: "nonprd"
    compliance: "iso27k"
    networkzone: "dmz"
```
- When we create the above namespace with name **thebank**, the name gets adjusted via Kyverno adding **-iso27k**.
- A new Kubernetes **network policy** has been created by Kyverno in the new namespace.
- The new Kubernetes **resource quota** has been applied to the namespace.
- An extra label **backupEnabled=true** has been added to our namespace.

![namespace]({{site.baseurl}}/assets/img/2021-06-02-namespace1.png)
![namespace]({{site.baseurl}}/assets/img/2021-06-02-namespace2.png)

# 5. Reporting

## 5.1 Terminal

Via kubectl we can get a basic report about the violations of each policy.
```
kubectl get polr --all-namespaces
kubectl describe polr --all-namespaces | grep -i "status: \+fail" -B10
```

## 5.2 Reporter UI

**[Policy reporter](https://github.com/fjogeleit/policy-reporter)** brings visiblity into Kyverno policy enforcement.

![reporter]({{site.baseurl}}/assets/img/2021-06-02-reporter.png)

*The configuration we used in this post can be found on <https://GitHub.com/dewolfs/kyverno-aks>.*