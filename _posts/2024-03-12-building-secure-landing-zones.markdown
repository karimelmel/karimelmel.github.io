---
layout: post
title:  "Building Secure Landing Zones"
date:   2024-03-12
---

Securing cloud environments becomes increasingly critical as more organizations embrace public cloud services. However, building secure cloud environments is not always straightforward, and organizations often face challenges in ensuring security, integration with existing processes, and adoption. One concept that can help address these challenges is the Landing Zone. 

Public cloud is typically introduced by accident, like an eager developer wanting to build something without having to wait for the server team or someone that wants to spearhead a modernization effort in a silo.

When organizations consciously decide to use the public cloud, questions often arise about whether it is secure, integrates with existing processes, and to what extent it will be adopted.

Regardless of which cloud provider you choose, you will be introduced to the concept of a ‘Landing Zone' early on.

In this post, we aim to elaborate on the concept of building Landing Zones and how to build them with security by design.

# The concept of Landing Zones

## Introduction to Landing Zones

The concept of a landing zone refers to an environment, a segment, and a trust boundary, all of which are supported by a centralized platform.

Imagine you have a new project. You want to create a public, customer-facing API to retrieve information. You've decided to use the cloud to take advantage of services with more abstraction and avoid the need to maintain servers. The data for this API is stored in a database operated by another party, and there is no centralized web server to host it, at least not one that meets your modern solution requirements.

Just signing up for one of the major cloud providers won't be enough. You will still need to establish network connectivity to the private database, monitor the application and management plane operations, and learn how to perform Continuous Integration.

This is where the landing zone concept comes in. Suppose you work for an organization that has embraced this concept. All you need is a service request and some to run an automation task to have the necessary capabilities for your application landing zone. In a Landing Zone, your application will be entirely disconnected by default. However, with mutual consent from the database owner, you can obtain the required access and start building your application securely.

## Platform and capabilities

To support the concept of Landing Zones, you will need a centralized component, often referred to as the Platform. The Platform ensures the provisioning (also known as ‘Vending’) of the Landing Zones. It also hosts central capabilities, such as private and hybrid connectivity, identity, ‘Guardrails,’ monitoring, and possibly more capabilities that will reduce your time to market.

# The security challenge with Landing Zones

While the idea of a Landing Zone is complete decentralization, security, and isolation, we frequently come across flawed implementations. Typically, this is due to shortcuts taken to introduce new platform capabilities, where the IT Architects behind the implementation neglected the importance of segmentation, or because the concept is complex and requires domain-specific expertise. It’s also important to keep in mind that the frameworks provided by the cloud providers for landing zones don’t include considerations for what we are covering here. 

We will seek to summarize the most common mistakes we come across and what some of the mitigation strategies are.

### Continuous Integration and Deployment

Landing Zones are often introduced with the notion of doing ‘everything as code,’ which usually goes wrong when people forget that the integrity of your code becomes vital for your infrastructure. The source code repository administrator can now compromise the entire platform and the landing zones, or a malicious actor compromising a developer account can gain control of your infrastructure through the source code. While this is trivial to mitigate, I rarely see organizations understand the importance of this risk.

For Continuous Deployment, we frequently encounter workloads where the source code used to be the truth, but where it’s no longer the case.  In many environments, the secure configuration has drifted, and there’s a day of work before the code becomes idempotent again.

A selection of essential questions you should ask yourself for Continuous Integration and Deployment:

1.   Keep in mind the four-eyes principle. 
    1. What are the ways around it? 
    2. Are there system admins that can bypass controls? 
    3. Are all repositories used for CI protected? 
    4. Do you have protection on the branch and is the service account accessible from a branch without protection?  
2. Are you regularly deploying to make sure your code is idempotent?
3. Does the code reflect the truth in the environment?`

### Cloud Identity hygiene

You might be used to guarding your traditional identities or ensuring strong authentication, but have you looked at the possible attack paths in your cloud identity system? Now that you rely on it for the most critical information, you should know which identities can modify what groups or update the credentials for what applications. We commonly encounter users with low privileges who can gain administrative access to the entire platform and landing zones. 

A selection of essential questions you should ask yourself on your Cloud Identity hygiene are:

1. How many administrators do you have?
    1. Direct administrators
    2. Indirect administrators, such as Group Owners, Service Principal Owners/App Owners
2. Are you actively keeping up to date with attack paths?
3. Are you using a secure device for administrative access?
4. Are your user accounts for admin tasks separate from your email account? 
5. Are you using FIDO2?

### Using Cloud Services safely

A lot of cloud platform consumers view themselves simply as consumers of a service. When setting up a Landing Zone, it is important that the services provided are limited to what the organization is comfortable with providing securely. If a Container solution seems appealing, it is important to consider whether it is something that should be supported as a platform capability. If not, we recommend restricting any usage of it to a sandbox to experiment, but avoid having it in production environment.

Once you have determined which services you want to use, you should establish a process for adding support for those services within your organization. This will help ensure that the services can be used securely and in accordance with the organization's policies and procedures.

A selection of essential questions you should ask yourself for making sure you are using cloud services safely are:

1. Do you have a list of the supported resources?
2. Do you have a process for onboarding new services to assess the security risks?
3. Do all the supported resources have logging and monitoring enabled?
4. Do you have guardrails to prevent insecure configuration?
5. Do you allow developers to consciously override the guardrail while documenting their exemption?
6. Are you aware of the risks associated with using the service?

# Summary

To conclude, Landing Zones are a crucial concept to build secure cloud environments. It provides an environment, a segment, and a trust boundary supported by a centralized platform that hosts central capabilities to reduce time to market. However, flawed implementations can compromise security and isolation, which is why there are things you must get right. Simply relying on a framework won’t get you there. If security is a priority, organizations will effectively build and maintain secure cloud environments through Landing Zones.
