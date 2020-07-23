---
title: Backup and DR for Data Platforms
titleSuffix: Microsoft Cloud Adoption Framework for Azure
description: Considerations for backup and DR of cloud data platforms
author: dalusty
ms.author: dalusty
ms.date: 07/29/2019
ms.topic: guide
ms.service: cloud-adoption-framework
ms.subservice: operate
---

# Introduction

This guide aims to show effective ways of backing up and recovering a cloud data platform, as well as how to recover in a regional outage or disaster recovery situation. Data loss prevention and business continuity can be extremely important once a data platform becomes line of business. Implementing this in a cost effective manner which will allow recovery with an impact appropriate to your business can be challenging. This guide will familiarise you with the concepts and trade offs and prepare you to design an effective solution.

## The Data Platform

In this instance cloud data platform refers to cloud data warehousing using data lake and Data Factory such as in the image below. While this is a specific solution, the concepts will translate well to other scenarios including realtime data processing scenarios using a data lake architecture.

![Data Platform Overview](images/DataMarts.png)

# Systems Of Record

## Backup

These data sources, such as existing SQL Databases, CRM, ERP and other systems should already have backups in place. They are generally line of business and the data is therefore neither transient nor trivial. As live systems it is possible that data corruption or mistakes can happen and so data will need to be protected.

## Disaster Recovery

A disaster recovery plan should already be in place for these data sources where appropriate. The target location of these recovery plans should be understood and taken into account when planning for data platform recovery.

# Ingest

## Backup

Generally, the ingest stage is transient and therefore there will not be any data to back up here. The scripts and configuration for this component should be backed up, however. As changes are made to scripts it is possible for configuration errors to be introduced which may need to be backed off. Since the ingest of data will change the data in the lake, longer term retention may not be required. Ideally a DevOps style approach using code management such as GitHub or an Azure DevOps code repository should be used to maintain code versioning and to deploy new versions in a controlled way. Alternatively change control can be used in more manual processes.

## Disaster Recovery

DR for ingest services can be complex and is split into two sides. There are on premises components such as data gateways which need to be protected in line with your other on premises systems. This protects against an outage of your on premises data centre. Secondly is the protection of the cloud components. These need to maintain connectivity to data sources from the recovery data centre or region. Since there is no data within the service, it is often simpler to redeploy and configure in DR than to maintain a primary and secondary instance.

# Store

There are usually various stages of data within the data lake such as raw, curated, enriched etc. and each of these may need to be treated separately based on business value and how long they might take to recreate.

![Data Staging](images/DataStaging.png)

 The definitions of these may also vary based on use-case. For instance raw data in an IoT scenario might be kept long term as a kind of “system of record” while in a batch system the raw incoming data is only needed until a curated copy is made and backed up.
The questions to ask here are:

* Can I recreate the data from source later?
* Will the source retain the data as long as I plan to keep this copy?
* Will it cost less to back up this data than to re-create it?
* Will it take longer to re-create than my RTO allows?

## Raw

Generally when data is copied into the lake it will be copied verbatim with no changes or very minimal changes. This means that there is no longer a requirement for the system of record to keep the data, and we no longer rely on that system to hold a copy. For this reason, the raw data store would generally be both backed up and have a disaster recovery set up.

## Enriched/Cleansed

Following raw data we would usually do some cleansing and modification. Here we will change schemas and data types to match that needed in the data lake, as well as joining different data sets together. These processes will all be fully automated and so no human interaction happens at this stage. Because of this, backup achieves very little. If the data is corrupt or incorrect, the process is at fault and so that is what needs to be fixed and re-run. Using permissions accidental deletion can be ruled out.
Disaster recovery for this stage will be useful since storage is cheap and compute is expensive. It would be rare for it to be cheaper to recreate this data than to simply store two replicated copies.

## Curated

Curated data has been prepared into our intended structure for consumption. This stage may well include processes that modify files and so backup could be a requirement to allow rollback of files. As above, a disaster recovery replicated copy may be used to reduce time in a disaster scenario. Use of partitioning may allow for smaller amounts of data loss, for instance weekly partitions may mean that only a week of data need to be reprocessed in a disaster.

## Presentation

In the presentation stage, data may be separated into data warehouses or data marts. Here we would certainly wish to store backups and a replicated copy for DR unless the warehouse system itself is replicated. Often in fast moving data the whole warehouse may be reloaded every day and so a full set of data might be stored for each day and backed up.

# Prep and Train

Prep and Train is used to model data either for warehousing or for machine learning purposes. This area can take one of two approaches; either an ad-hoc experimental approach, or an industrialised and tested approach. Generally you will end up with two stores here, one for experimentation early on in the project lifecycle, or for ongoing analysis and the other for refined processes. These areas can be treated differently for backup and DR purposes and it's likely that the RPO and RTO for each is going to be different.

For the industrialised processes, you will usually have more strict SLAs in place since other processes and systems will almost certainly rely on them. Scripts and systems will change less often and will usually be managed through change control (e.g. ITIL) or DevOps (e.g. Agile).

For the experimental processes things may be a bit more ad hoc and so backup might need to be in the hands of users such as data scientists rather than defined processes and technology. With new pipelines being created more often it is much harder to enforce backups.

## Scripts

Generally speaking, scripts will be stored in a Git repository such as GitHub. This will allow for commits to be used as snapshots for backup since a commit is needed any time an item changes. Using this technique, branches and merges may also be used to prevent issues from forming, and the new branch applied to a development copy of the platform prior to being merged into production.
Git repositories can be cloned to another host for disaster recovery, although you will need to ensure that at least one copy is blocked from deletion to protect against accidental human error removing the repository.

## Modeled Data

When data has been processed and modeled its value will increase. There should be a backup of this data since it is likely to be a direct contributor to the final consumption of the information. Modeled data may be processed and stored as files on the data lake. In this scenario, it is common to see the data lake used as a form of backup by creating a new and complete set of data each time an update is completed. This means that any complete data set can be loaded at any time to recover to a point in time. Alternatively, if the data is loaded into a semantic layer system, that system may provide for point in time recoveries for the data natively.

## Infrastructure Configurations

The prep and train stage often has related infrastructure and systems. These may be backed up using traditional methods if they are IaaS based. When using PaaS or SaaS solutions here, a backup of the configuration and deployment script would allow recreation of the environment in a recovery scenario, or following a change that has broken the solution. Many solutions here will have JSON based configurations and deployment scripts and so should be relatively easy to back up using a solution such as Git alongside the scripts.

# Model

Within the model layer of the solution, data is generally ingested into a product where it can be queried. The decision of whether to back this up will depend on how data is generally ingested. Here, we have two options, often based on load times as to which is chosen:

* Daily full ingest
* Paritioned incremental

In situations where a full daily ingest is carried out with all data, there is no reason to back up or replicate the system you're loading into. Here, you would simply create a folder on the data lake with a copy of the data for each load and this would constitute a full backup for the load. For disaster recovery, this data can either be loaded to a secondary system, or left on a replicated data lake ready to load in the event of a failover. The decision as to which would simply be a comparison of the load time with the RTO, taking into account any time to act following a disaster event.
Where partitioned incremental loads take place, it might be necessary to back up in several places. Firstly you would need backups of the incremental data, since each partition might be loaded with different data each day and so it may be desirable to recover a previous version. Secondly you may wish to backup within the solution itself, in order to roll back from a failed load. For disaster recovery you will need to replicate the data lake containing files to load. To shorten load times you may also wish to replicate the data solution itself to another location.

# Serve

Your serving layer may be extremely varied in terms of technology. Here we will assume you have caching and presentation (reporting) layers.

## Cache

A caching layer will allow for many more users to access the data in parallel, at the cost of some flexibility. Since data will be reloaded every day it is unlikely that you will need to back up this layer. Data will be available from the modeling layer and so can be rolled back there if necessary. For disaster recovery it is likely that you would backup the deployment and configuration scripts in Git and redeploy them before reloading data. This may be product dependent, and so in some scenarios a backup of the cached data may be advisable.

## Presentation Layer

The presentation layer can vary widely. Here we will assume a reporting platfrom. In this scenario, the report source should be backed up and version controlled using a supported technology such as Git. For Disaster recovery, you may need to redeploy reports to another reporting platform in another region, and potentially re-point any data sources to the new location. It might be possible to use technology such as global load balancers (in Azure this would be Traffic Manager) to handle URI changes between regions during a failover.
