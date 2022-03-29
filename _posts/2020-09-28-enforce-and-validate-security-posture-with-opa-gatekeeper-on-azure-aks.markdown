---
layout: post
title: Enforce and validate security posture with OPA Gatekeeper on Azure AKS
date: 2020-09-28 14:30:20 +0300
description: Enforce and validate security posture with OPA Gatekeeper on Azure AKS # Add post description (optional)
img: 2020-09-28-open-policy-agent.jpg # Add image post (optional)
fig-caption: Photo By Valiphotos on Pexels # Add figcaption (optional)
tags: [Kubernetes, Azure, OPA, Gatekeeper, AKS]
---

Compliance, governance, and security are nonfunctional requirements that every system needs to satisfy. Kubernetes clusters are no different.  We need something to build preventative controls to stop unwanted changes in our clusters. We can also shift the controls left, into the our CI/CD automation, evaluating changes before they are pushed.

# 1. Introduction

![image]({{site.baseurl}}/assets/img/2020-09-28-logo.png){: style="float: left"}

Organizations need the ability to apply rules to their workloads and services, at scale and distinct from the development of those services. Policies and policy enablement provide those governance capabilities with declarative approaches.  **[Open Policy Agent (OPA) Gatekeeper](https://github.com/open-policy-agent/gatekeeper)** integrates with Kubernetes and is able to provide the right guardrails to enforce structure and keep your deployments running smoothly. 

Gatekeeper brings a structured schematized version via constraint templates and constraints using Kubernetes.  Rego is the language that OPA and Gatekeeper support to validate a request to the Kubernetes cluster.

Policies are essential to the long-term success of an organization, because they encode important knowledge about how to comply with legal requirements, avoid repeating mistakes, and so on.

For example, a custom policy could be that developers are ONLY allowed to reference container images in their deployments from your own private registry. Other could be, Developers must have certain labels be present in all deployment definitions identifying the business unit to chargeback to.

![install]({{site.baseurl}}/assets/img/2020-09-28-gatekeeper-design.png)

*Credits: [https://kubernetes.io/blog/2019/08/06/opa-gatekeeper-policy-and-governance-for-kubernetes](https://kubernetes.io/blog/2019/08/06/opa-gatekeeper-policy-and-governance-for-kubernetes/)*

A constraint template contains the Rego signature, it basically contains all the logic of what happens when.  Gatekeeper executes these constraints.  If the rule matches, the constraint is violated.

Gatekeeper implements Open Policy Agent (OPA) as a set of Kubernetes Custom Resource Definitions (CRDs).  The CRDs are watched by OPA via Gatekeeper and it watches through the API server all of the objects that gets created through the Kubernetes API server.  This is a cloud-native way of enforcing policies.

# 2. Installation

To install the latest version of Gatekeeper, run the following command:
```
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```
The following Kubernetes objects are created:

![install]({{site.baseurl}}/assets/img/2020-09-28-install.gif)

In the logs of the gatekeeper-controller-manager container you can find back the startup of the different roles:
```
kubectl logs -f gatekeeper-controller-manager-6dff56479c-46sr8 -n gatekeeper-system
```

Part of the installation is the admission control webhook.

```
kubectl get validatingwebhookconfiguration
kubectl get validatingwebhookconfiguration gatekeeper-validating-webhook-configuration -o yaml
```
The configuration of the webhook intercepts any **create** or **update** operation on every **apiGroup** of every **apiVersion** on every **resource**.  All these actions will be passed to the **gatekeeper-webhook-service**.

![webhook]({{site.baseurl}}/assets/img/2020-09-28-webhook.gif)

# 3. Example 1 - Required labels on namespace

Now that we have Gatekeeper installed, we will try out a few good and bad configurations.  For our first example we want to enforce that labels are applied to the Kubernetes namespace. At the end of this blogpost you can find the link to the repo containing all the code used in this blogpost.

## 3.1 Constraint template

We have defined a constraint template which contains the Rego policy to verify the existance of Kubernetes labels.

```
kubectl apply -f template/k8srequiredlabels_template.yaml
```

To view all the constraint templates and the individual details use the following commands:

```
kubectl get constrainttemplates.templates.gatekeeper.sh -n gatekeeper-system
kubectl get constrainttemplates k8srequiredlabels -n gatekeeper-system -o yaml
```

## 3.2 Constraint

The constraint itself contains the parameters that are used in collaboration with the constraint template.

```
kubectl apply -f constraint/owner_must_be_provided.yaml
```

## 3.3 Results

Now that we have our Gatekeeper constraint in place, we are going to create a new Kubernetes namespace without any label.  The error message is clearly stating why the deployment is failed and also which constraint is blocking this.

![webhook]({{site.baseurl}}/assets/img/2020-09-28-label-bad.gif)

If we provide a Kubernetes manifest with a label, the namespace gets created without errors.

![webhook]({{site.baseurl}}/assets/img/2020-09-28-label-good.gif)

Now that we have the constraint in-place inside our Kubernetes cluster we want to get some knowledge on how many violations we count.  The constraint contains a violations section, describing the namespaces that are not compliant.

```
kubectl describe k8srequiredlabels.constraints.gatekeeper.sh 
```

![webhook]({{site.baseurl}}/assets/img/2020-09-28-label-violations.gif)

# 4. Example 2 - Allowed container repository

In the second example we will configure a policy to only allow pull container images from a known container registry.

## 4.1 Template

First we deploy the constraint via the following command:

```
kubectl apply -f template/k8sallowedrepos_template.yaml
```
To view the constraint templates and the individual details use the following commands:

```
kubectl get constrainttemplates.templates.gatekeeper.sh -n gatekeeper-system
kubectl get constrainttemplates k8sallowedrepos -n gatekeeper-system -o yaml
```

## 4.2 Constraint

The constraint itself contains the parameters that are used in collaboration with the constraint template.

```
kubectl apply -f constraint/limit_repo.yaml
```

## 4.3 Results

When we launch the Kubernetes deployment which is referencing a public container repository, it wil get created without error.  However, when we verify our running pods in the production namespace, we can't find any pods. When we verify the events in the namespace, we understand that the container repository defined in our Kubernetes manifest is not allowed.

![webhook]({{site.baseurl}}/assets/img/2020-09-28-repo-bad.gif)

If we now execute a deployment where we reference to a private container repository allowed by our constraint, our deployments gets successfully deployed and our pod are also created.

![webhook]({{site.baseurl}}/assets/img/2020-09-28-repo-good.gif)

Looking to the details of the allowed repository constraint we can see how many violations exist.

```
kubectl describe k8sallowedrepos.constraints.gatekeeper.sh 
```
## 5. Example 3 - Container limits

For our third example we want to make sure that each Kubernetes deployment has container resource limits set in the manifest file.

## 5.1 Template

First we deploy the constraint via the following command:

```
kubectl apply -f template/k8scontainerlimits_template.yaml
```
To view the constraint templates and the individual details use the following commands:

```
kubectl get constrainttemplates.templates.gatekeeper.sh -n gatekeeper-system
kubectl get constrainttemplates k8scontainerlimits -n gatekeeper-system -o yaml
```

## 5.2 Constraint

The constraint itself contains the parameters that are used in collaboration with the constraint template.

```
kubectl apply -f constraint/containers_must_be_limited.yaml
```
## 5.3 Results

First we apply a Kubernetes deployment with no container resource limits.  The error message clearly indicates what the problem is.

![webhook]({{site.baseurl}}/assets/img/2020-09-28-limit-bad.gif)

If we now execute a deployment with container resource limits, the deployment and pod get deployment without a problem.

![webhook]({{site.baseurl}}/assets/img/2020-09-28-limit-good.gif)

# 6. Dryrun

If we have a brownfield cluster with a lot of resources and we want to bring the cluster into compliance, we can start with the enforcement action of dryrun.  This will allows us to get visibility and pass through all policies resulting in finding out what is out of compliance and take action to fix things.  
Add **constraints.spec.enforcementAction** to enable dryrun and to see the state of compliance of the cluster.

![dryrun]({{site.baseurl}}/assets/img/2020-09-28-dryrun.gif)

# 7. Monitoring

Prometheus is supported as backend for metrics of Gatekeeper.  There are metrics for:
- violations per enforcement action
- last audit timestamp
- audit duration
- total number of constraint templates and constraints

In order to enable Prometheus to scrape the metrics of Gatekeeper, we have to add the following Kubernetes annotations to the deployment of the **controller manager**.
```
kubectl annotate deploy gatekeeper-controller-manager -n gatekeeper-system prometheus.io/port=8888
kubectl annotate deploy gatekeeper-controller-manager -n gatekeeper-system prometheus.io/scrape=true
kubectl annotate deploy gatekeeper-controller-manager -n gatekeeper-system container.seccomp.security.alpha.kubernetes.io/manager=runtime/default
```

We repeat the same operation for the **audit** deployment of Gatekeeper.
```
kubectl annotate deploy gatekeeper-audit -n gatekeeper-system prometheus.io/port=8888
kubectl annotate deploy gatekeeper-audit -n gatekeeper-system prometheus.io/scrape=true
kubectl annotate deploy gatekeeper-audit -n gatekeeper-system container.seccomp.security.alpha.kubernetes.io/manager=runtime/default
```

As last, we need to add the targets of Gatekeeper to the Prometheus configuration.  The Prometheus configration can be adjusted via the Kubernetes configmap.

```
kubectl edit cm prometheus-server
```

Add the following code block to the static configuration file.
```
    - job_name: gkaudit
      static_configs:
      - targets:
        - <gatekeeper_audit_service_ip>:8888
    - job_name: gkmanager
      static_configs:
      - targets:
        - <gatekeeper_manager_service_ip>:8888
```  

When we open the Prometheus UI, we are able to select the Gatekeeper metrics.  Prometheus Alertmanager can be configured to send a notification when a threshold is breached.

![prometheus]({{site.baseurl}}/assets/img/2020-09-28-prometheus.gif)

# 8. Conftest in CI/CD

Gatekeeper templates and constraints make sure that requests are validated against Rego policies before anything happens inside your Kubernetes cluster.  Additionally we can make sure that we already scan the Kubernetes manifests in our repository to verify if a violation would occur at deployment time.

[Conftest](https://www.conftest.dev/) helps you write tests against structured configuration data.

In our repository, we have defined an Azure DevOps pipeline **1-conftest.yml** which runs on a scheduled daily basis during the night.  The pipeline contains executes a scan of the Kubernetes manifests against the Rego policies.

![pipeline-fail]({{site.baseurl}}/assets/img/2020-09-28-pipeline-fail.gif)

![pipeline-success]({{site.baseurl}}/assets/img/2020-09-28-pipeline-success.gif)

*The configuration we used in this post can be found on <https://github.com/dewolfs/opa-gatekeeper-conftest>.*