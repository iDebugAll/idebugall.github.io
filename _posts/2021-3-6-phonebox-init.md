---
layout: post
title: "NetBox as Voice and UC Source of Truth"
---

Have you ever been struggling with an enterprise Voice and Unified Communications infrastructure 
documentation?

- What phone number is that? Where it comes from?
- Is this SIP-trunk relevant?
- Which of these Excel files contains the information I need?
- Do we have a spare phone number for a new service?
- Phone_numbers_latest_201907(3).xlsx?!

Sounds painfully familiar? I have got something that might help.

<cut/>

TLDR: I developed a [new plugin](https://github.com/iDebugAll/phonebox_plugin) for [NetBox](https://github.com/netbox-community/netbox) to manage phone numbers and more. Using NetBox as Voice and UC Source of Truth might be a good idea.


## How and What Enterprises Document

I have seen many examples of how the documentation can be organized since 2011 when I started my career. I've been managing and consulting on Voice and UC deployments serving thousands of users and consisting of hundreds of voice devices and circuits in total. However, regardless of the region and size, they all shared one thing in common: the documentation related to Voice and UC relied on Microsoft Office and PDF files stored in a more or less structured fashion.

It is a good question, how representative this sample is. Based on discussions with other engineers, this is quite typical. Dedicated domain-specific documenting solutions for Voice and UC are not that common in the enterprise sector. "What is the problem with that?" you may ask. Let's try to figure this out.

To begin with, I would break the pieces of the information we are usually interested in and keeping track of as follows:

1. **Solution design documents and implementation summaries.**
   *How it was designed and expected to work*.
   You can typically find such files in most organizations.
2. **Network and domain-specific diagrams.**
   *How it looks visually*.
   This might be a part of #1.
3. **Inventory information.**
   *What physical and virtual boxes we have*: Voice and UC equipment, virtual machines, PRI and DSP modules, etc. Related models, part IDs, serial numbers typically fall into this category.
4. **Cabling and device interconnection details.**
   *Cable paths across patch panels and between ports*.
   Those who spent hours with a [tone generator and wire tracker](https://www.amazon.com/Generator-Accuracy-Inductive-Amplifier-Adjustable/dp/B08DV1N2Q9/) trying to identify the actual cross-connections from a traditional PBX will likely sigh at this point. *\*Sighs\**. In VoIP environments, this information is still useful to know.
6. **IP addressing.**
   *How we assign IP-addresses to our devices*.
6. **Voice Circuits.**
   *What voice interconnections we have*: SIP-trunks, PRIs, CO lines.
   It is necessary to know what their requirements are and how many lines they offer.
7. **Phone numbers.**
   *What PSTN pools and internal extensions we have*.
8. **Call routing.**
   *How we route phone numbers and patterns*.
9. **Configuration details and templates.**
   *What configuration elements translate #1 into a working infrastructure*.
   The actual configurations on the devices and VMs are often the only source of such information.
10. **Management access details.**
    *How do we get access to our infrastructure elements*.
11. **External contracts and agreements.**
    *What external services we have and what we billed for*.
    Searching phone numbers through PDF scans of the service provider agreements is adventurous. *\*Sighs again\*.*

The list demonstrates some general categories. It might change from one organization to another.
There is no single format convention nor standard templates across different organizations.
Furthermore, some additional challenges arise with regular text files and spreadsheets:

1. There are no clear relations between each piece of the information. Especially cross-document.
2. It is hard to search and index the information across distributed files.
3. It heavily relies on your knowledge of each file's location and purpose.
4. Some information may be duplicated across the documents.
5. It is hard to validate the information and compare it with the actual infrastructure state.
6. Reporting requires extra effort and manual data transformation and analysis.
7. A whole picture may be missing. UC and Network teams may operate independently. The larger organization is, the higher chances are. Cross-functional tasks like setting up and maintaining Voice QoS may become much harder in such cases.


The information we store in our documentation should be up-to-date and consistent to help us operate our infrastructures reliably and efficiently. *Some* documentation is usually better than no documentation at all. However, blindly trusting the documentation may lead to even worse scenarios. The need to double-check the actual state every time eventually pushes us to avoid opening to the documentation and forget to update it afterwards. As long as the configuration changes can be applied unreflected in the documentation, sooner or later, they will.

Text files and spreadsheets are not necessarily a poor choice. General design documents are more self-sufficient and tend to stay valid in a long-term perspective. Regular file formats are typically acceptable and suitable for them. Frequently updated operational documentation, in contrast, requires different approaches to overcome the limitations listed above.

## UC Infrastructure-as-Code and UC Source-of-Truth

One of the solutions the industry has come to is Infrastructure-as-Code (IaC) paired with Single-Source of-Truth. It redefines the idea of how we treat our infrastructures and perform changes:

- An infrastructure representation moves toward machine-readable scripted or declarative configurations. Using this approach, you can apply the advantages and best practices of software development, testing, and DevOps.
- Single Source of Truth (or simply Source-of-Truth, SoT) is an idea of identifying a single place for each piece of the information about our infrastructure where it can be found and must be managed. It does not mean all the data has to be stored in a single place. It also does not assume the individual pieces of data can not exist in multiple copies anymore. The key requirement is that the primary source for each piece of data must be unique.
- Changes are applied to the infrastructure components by the teams indirectly. You change the data in your Souce-of-Truth first. Then it gets validated, applied to the IaC model, tested, and eventually delivered to the target devices and VMs by the Automation stack of your choice.

Souce-of-Truth contains a desirable state of your infrastructure. The purpose of the automation stack is to bring the actual configurations in compliance with the Source-of-Truth based on Infrastructure object models you store as code and scripts. It eliminates the problem of inconsistent documentation as now you change your infrastructure by updating the documentation. 

It is vital to notice that moving from the traditional workflows to the Source-of-Trust and Infrastructure-as-Code does not has to be done in a single breaking step. It is possible to migrate the processes and procedures gradually. Rebuilding the documentation framework might be a good starting point.

These concepts are already widely used by DevOps engineers.<br/>
Being applied to Networks, NetDevOps has greatly evolved during the past few years.<br/>
I believe all these practices are applicable to the core UC domain as well. Not to mention that UC components might even share the platforms with regular Network functions. Initial provisioning steps for an SBC and a Router or a Switch are very similar. Automating BGP peering is not too far from automating SIP trunking.<br/>
Many best practices and tools from the NetDevOps are transferable to the UC domain. One of such tools is NetBox.


## Why NetBox

An obvious question is "What is NetBox?" A comprehensive answer from its developers:

>NetBox is an open source web application designed to help manage and document computer networks. Initially conceived by the network engineering team at DigitalOcean, NetBox was developed specifically to address the needs of network and infrastructure engineers. It encompasses the following aspects of network management:
>
> - **IP address management (IPAM)** - IP networks and addresses, VRFs, and VLANs
> - **Equipment racks** - Organized by group and site
> - **Devices** - Types of devices and where they are installed
> - **Connections** - Network, console, and power connections among devices
> - **Virtualization** - Virtual machines and clusters
> - **Data circuits** - Long-haul communications circuits and providers
> - **Secrets** - Encrypted storage of sensitive credentials

NetBox is designed as the Network Source-of-Truth. Everything in NetBox is an object, and *almost* everything is accessible via API that provides great opportunities for NetBox integration with external systems. There is a relational database and data model behind NetBox objects. NetBox is well [documented](https://netbox.readthedocs.io/en/stable/) and has great [community](https://join.slack.com/t/netdev-community/shared_invite/zt-mtts8g0n-Sm6Wutn62q_M4OdsaIycrQ) around it. Not suprisingly, the popularity of NetBox keep growing worldwide.

You might have noticed that the list above already covers several categories from the list of Voice and UC documentation. PBXs, SBCs, Gateways, MCUs, and many other Voice and UC boxes are **Devices**. You can usually find them installed into the **Equipment Racks** below the ToR switches. They are interconnected with **Connections** and may terminate some **Data Circuits** carrying Voice signaling and media from the Telephony Service **Providers**. Some of Voice and UC functions may be running as **Virtual Machines**. And all (unless you are using electromechanical PBXs) Voice and UC infrastructure components use **IP-addresses**.

But some critical points like **Phone Numbers**, ***Voice*** **Circuits**, and **Call Routing** are missing. Fortunately, NetBox provides extremely powerful extensibility feature called [Plugins](https://netbox.readthedocs.io/en/stable/plugins/). It allows you to extend the core NetBox data schema and embed new features. Combined with native NetBox data models, such a plugin would provide you with a full set of necessary abstractions to describe your Voice and UC deployment.

So I introduce the [PhoneBox](https://github.com/iDebugAll/phonebox_plugin) plugin for NetBox.

## PhoneBox Plugin

This plugin is intended to bridge the gap between general NetBox network data models and those of Voice and UC world.
I started development recently and already implemented **Phone Numbers**. It provides some instant value and allows you to manage PSTN DIDs and internal extensions using NetBox. Some [feature requests](https://github.com/netbox-community/netbox/issues/1893) from the NetBox community to add support for such abstractions were already there.

Each Phone Number consists of the following attributes:

- Number – An individual phone number.
- Tenant – A mandatory relation to Netbox *Tenant* object. It allows for number plan partitioning.
- Description – An optional text description for the number.
- Provider – An optional relation to NetBox native *Provider* object. It identifies the provider of the number.
- Region – An optional relation to NetBox native *Region*. It represents the geographic location of the number.
- Forward_To – an optional relation to another Number object. It represents a Call Forwarding scenario.
- Tags – An optional relation to NetBox native tag objects.

The plugin implements all Create, Read, Update, Delete (CRUD) operations for the Phone Numbers via NetBox web interface and REST API. 
It also supports bulk import from CSV for the Phone Numbers to simplify the migration of your data.

I am planning to add more Voice and UC abstractions to the plugin in the future.
The complexity of selecting proper data models that fit into arbitrary infrastructure is hard to underestimate. I am going to write a separate post about it.

Please feel free to leave the comments below or to reach me on social media. Your feedback matters.
