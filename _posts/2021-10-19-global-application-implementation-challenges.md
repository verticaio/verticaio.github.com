---
layout: post
title: 'Global application implementation challenges: from security to resiliency and low latency'
date: 2021-10-19 17:30:20 +0300
description: 'Global application implementation challenges: from security to resiliency and low latency' # Add post description (optional)
img: 2021-10-19-frontdoor.jpg # Add image post (optional)
fig-caption: Photo By Quang Nguyen Vinh on Pexels # Add figcaption (optional)
tags: [ Azure, Bicep, cdn, waf, security, Front Door, appservice, web, owasp]
---

The non-functional requirements around the performance enveloppe (low latency, high availability and high service level agreements) are not getting any easier nowadays.

Compute and storage units tend to be co-located or close to each other for latency and performance reasons.  From an architectural viewpoint, the geodes pattern is providing the right guidance.

![geodes]({{site.baseurl}}/assets/img/2021-10-19-geode-dist.png)
Source:  **[Microsoft](https://docs.microsoft.com/en-us/azure/architecture/patterns/geodesn)**

Multiple geodes consist of a collection of types of resources that are co-located close to each other and stamped around different geographies.
Data is replicated among the different regions using a replication backplane together with traffic routing or edge network acceleration technology in front off them to present all of the geodes globally as a single system.  No geodes should be dependent on another and shared components should be globally fault-tolerant.

Azure Front Door plays a crucial role in the geodes pattern.

In this blogpost, we describe how to use the new Azure Front Door premium (preview) with private origin support.

# 1. Introduction

**Azure Front Door Premium** is a fast, reliable and secure modern cloud CDN that uses the Microsoft global edge network and integrates with intelligent threat protection. It combines the capabilities of Azure Front Door, Azure Content Delivery Network (CDN) standard and Azure Web Application Firewall (WAF) into a single secure cloud CDN platform.  

The Azure Private Link integration is an industry first CDN capability that enables customers to keep their origins private and embrace a zero-trust access model.  This integration removes the necessity of having origins with public internet accessible IP addresses, thereby significantly reducing the surface area. 

It is **the** solution to protect web applications without compromising on delivery speed.
The image below gives an overview of what we will implement today.
![Front Door]({{site.baseurl}}/assets/img/2021-10-19-design.png)

# 2. Configuration

Once we have our resources defined in Bicep (see link to Github repository below) and deployed via Azure DevOps pipeline we take a look at the Azure Front Door Premium (Preview).  The interface/configuration options are quite similar as the current Azure Front Door we all know.
- 1) Domains - DNS to access our applications.
- 2) Origin Groups - The backend targets.
- 3) Routes - how do we access the backend targets.
   
![Front Door]({{site.baseurl}}/assets/img/2021-10-19-frontdoor.gif)

The **Origin** pane contains configuration options about our backends.  Uptil now Azure Front Door was only able to connect to public endpoints.  The Premium SKU provides us now the capability to route traffic to private endpoints with the use of private link.

![fd-origin]({{site.baseurl}}/assets/img/2021-10-19-fd-origin.gif)

**!! When you make changes to the Azure Front Door rules/configuration, it takes +-10 mins to get the changes send to all the **[Front Door edge locations](https://docs.microsoft.com/en-us/azure/FrontDoor/edge-locations-by-region)** around the world!!**

## 2.1 Private Link configuration

After a successful run of our Azure DevOps pipeline, we try to access our Azure WebApp via the public URL (https://web-p-we-001.azurewebsites.net).  As expected we do get a 403 error.  
Our WebApp has private endpoints enabled which makes it not accessible on a public domain.

![webapp]({{site.baseurl}}/assets/img/2021-10-19-webapp-403.gif)

There is one more configuration which is required! 
Our Azure Front Door Bicep configuration contains a setting to enable the private link service. We need to approve the Private link connection request. Open the Private Link Center and approve the pending connection.

![private-link-approve]({{site.baseurl}}/assets/img/2021-10-19-private-link-approve.gif)

The result is quickly noticable when browsing to our Azure Front Door URL.

![results]({{site.baseurl}}/assets/img/2021-10-19-result.gif)

# 3. Monitoring

Since Azure Front Door is sitting in front of all our website instances (=origins), it has first hand knowledge of who is trying to access our websites.  Part of our Bicep configuration is sending the Azure Front Door Diagnostics to an Azure Log Analytics Workspace.  After hitting our Azure WebApp via the Azure Front Door URL we can consult the logs for additional data.  The logs provide us information about the audience who are trying to access our WebApp.

Use the following KUSTO query as a starting point:

```
AzureDiagnostics 
| where ResourceProvider == "MICROSOFT.CDN" and Category == "FrontDoorAccessLog"
| project TimeGenerated, requestUri_s, httpStatusCode_s, pop_s, clientCountry_s,originName_s
```

We can clearly identify the Azure Front Door POP of Belgium is being accessed.

![fd-logging]({{site.baseurl}}/assets/img/2021-10-19-fd-law.gif)

Sometimes it isn't possible for an application to log all this kind of information because they are under heavy load.  As a consequence you will miss this critical information.  This is one of the advantages of the extensive logging capabilities made available to us.

# 4. Geo-Location restriction

In order to reduce our attack surface we would like to block access to our website from specific regional locations around the world.  In this case we want to block traffic originating from Ireland.  
This can be done by creating an Azure WAF policy attached to our Azure Front Door resource.

Our Azure WAF policy should contain a custom rule with the following configuration:

![waf-block]({{site.baseurl}}/assets/img/2021-10-19-waf-block.gif)  

Custom rules are processed first and function according to the logic we select. This makes them very powerful as the first line of defense for web applications.

To verify our geo-location restriction we could spin up an Azure virtual machine in the Ireland Azure region or we can use Geopeeker.  
**[Geopeeker](https://www.geopeeker.com/)** is a tool for quickly viewing a site from different geographic locations around the world. 

Below you can view the outcome and conclude that our WebApp is being blocked from viewing when originating from Ireland.  
![geopeeker]({{site.baseurl}}/assets/img/2021-10-19-geopeeker.gif)

*The configuration we used in this post can be found on <https://GitHub.com/dewolfs/frontdoor-private-origin>.*
