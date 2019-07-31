---
title: Backup and DR Introduction
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

# Data Protection Concepts

## SLA

A good SLA is the start of any recovery solution. Ideally the SLA should be known prior to designing an availability solution. At one end of the scale would be a solution where no recovery is needed since re-implementation could be fast enough for the business use. At the other end of the scale, a sales organisation may depend on reporting and analytics to the point that money and revenue will be lost if the data platform is unavailable for any length of time.

**RPO**
The Recovery Point Objective. This is the amount of data loss you're prepared to accept. In large data solutions it is likely that a day of data loss is acceptable since it can take time to replicate data to another location. We would then re-process the lost data to catch up on the DR site. For other systems it may be that no data loss is acceptable and so syncronous replication is necessary which may add cost and complexity.
Whilst the data to be recovered from the primary site may allow for a more relaxed RPO, you should bear in mind that the actual objective for the recovered system might be to provide a more up to date system than the one being recovered. This is because reporting and analytics are constantly moving targets. Recovering a reporting solution exactly as it was 6 hours ago is not of any use when the report itself has a refresh of 4 hours. As such, it might be necessary to plan for processing some new data at the same time that older data is being recovered from backup or DR systems.

Flexibility may be used here to reduce costs. If the _really_ important information is the last four hours, then your recovery may set one RPO/RTO for that information and a different one for historic data. This may be sufficient to show the consumers of the information what they need in the short term before getting everything back later on.

**RTO**
The Recovery Time Objective is the amount of time allowed before recovery is complete. For data platforms this will vary depending on usage. Longer RTOs are preferable since they allow reduction of cost and complexity.

For the data platform, RTO and RPO may be complicated due to the pipeline nature of the data flows. If a Power BI report must have up to date data within 4 hours, then all systems bringing that data into the report must also be recoverable within that time, and complete any necessary processing to catch up data processing. This may not be achievable so be careful when writing SLAs for the various components in your solution.

## Backup and Restore

Backup involves creating recovery points which can be used to restore data to a previous version should anything go wrong. Backup does not require data to be copied elsewhere or taken off site, but this is often done. Where data is copied to a secondary location, this is considered a disaster recovery solution for the backup system, not part of the backup system itself.
Cloud platform backup and recovery is often a big change to traditional off-site tape backup. Firstly, we need to be clear of the purposes of the backup. The traditional tape solution was used for everything since it was the only avenue to any recovery. As such it included recovering data, recovering systems, and recovering during a DR scenario. In the cloud, the connected nature of the platform and the on-demand provisioning change this viewpoint. Since we are now able to provision on demand it is generally preferable to store the deployment configuration rather than a full system image. Since we have multiple, well connected regions it is preferable to have a cold, warm, or hot standby site rather than recover from scratch in a disaster. As such, backup may be seen as primarily a way to recover data following corruption or accidental modification.

Scenarios to consider for backup are:

* Accidental data modification
* Data corruption
* Accidental configuration change

Retention of backups as a form of archival should be avoided unless specifically called for in regulation. As a rule, backup copies do not aide compliance and in cases such as GDPR will often harm compliance. Archival of data will not be covered in this document and is considered a separate subject.

## Disaster Recovery

Disaster recovery involves recovering an entire solution in a secondary location following loss of the primary site. Traditionally this would have required a second copy of the system (server and networking etc.) and the data in the secondary location.

In cloud solutions, disaster recovery is sometimes very different to on-premises. Here, considerations such as speed of deployment and cost may affect the decision as to how you will recover. More than ever, the RPO and RTO requirements must be realistic and specific to the scenario. Where data is available in a replicated storage account, the difference between live continuity using clustering, booting a cold system, and building a new system from scratch might be around 30 minutes but many thousands of dollars in consumption. It is imperative, therefore to determine whether that 30 minute window while a database server deploys are worth the cost of leaving such a system running. In data platforms especially it is likely the workload is batch and would need to re-start a processing job anyway. Additionally, the likelihood of a regional outage from a global cloud provider is lower than that of on-premises solutions. The security and change controls alone make accidental outages less likely while advanced facility design and provisioning ensure that failure is less likely and may be mitigated rather than allowing a failure. Design of the platform ensures that many components within region are resilient or can be swapped out more quickly than a full DR failover. Again, the RTO comes into play as we need to balance the speed and complexity of a failover with the speed and efficiency of the cloud provider recovering. Data should always be made available in a secondary region, or saved to a location off-cloud. In the case of data platforms, much of the data is build from systems of record anyway and so can potentially be recreated rather than restored. Again, cost will come into the equation as you need to determine whether it’s cheaper to recreate and reprocess data or to recover it after storing backup copies.

Because of the above, we would generally separate the concepts of data recovery and system recovery for cloud DR. System recovery may be a script to install new systems, while data will need to be replicated or copied fresh in the recovery systems.

Scenarios to consider for cloud disaster recovery are:

* Extended Regional Outage
* Administrative errors
* Region changes (planned moves to a new region)

## Business Continuity

Business continuity performs the same role as disaster recovery above, but is designed in such a way as to ensure there is no loss of service when a region becomes unavailable for any reason. These scenarios will generally cost significantly more, or be more complex and so should be implemented only where a genuine requirement is defined.

## Archive

Data platforms are often used for longer term retention of information which may have been removed from systems of record. For instance, consolidated sales information may be kept long term in a data platform even when the original data is removed. In this scenario, treat the archive data as a system in its own right and not as a backup of source data. This means you should take backups or snapshots of the archive data, as well as potentially replicating the archive to a recovery site.
Archival of data will generally be either for compliance, or for historical data purposes. Before creating an archive you should have a clear reason for keeping data, as with all lifecycle management processes. Also ensure that you understand when the data will be removed and put in place processes to remove it at that time.
![Archive](images/archive.png)

Scenarios to consider archiving are:

* Regulatory Compliance
* Historical data for analytics
* Reduced cost/impact on systems of record

# Design Considerations

## Cloud Compute

Understanding how to create a suitable strategy in the cloud requires an understanding of cloud compute itself.
Differences to on-premises solutions are:

* Cost is metered rather than fixed
* Deployment can be instant rather than planned
* Storage can be cheaper
* Cloud systems can scale down and up easily
* Globally distributed data centres already exist

Your strategy will be a mixture of requirements and cost/performance. In the cloud, compute tends to be expensive while storage is cheap relatively speaking. As such it may often be more suitable to store multiple full and complete copies of data rather than incremental backup, or recreating data. For instance, with a data warehouse it might be that you choose not to back it up at all. Instead you may create a complete new set of tables in your storage every day and do a complete load daily. With this setup, a recovery would be as simple as reloading from a given set of files. Since storage is relatively cheap there is no problem keeping a month of daily versions of the full data set in most instances.

## Cost Time Tradeoff

When dealing with data recovery there is a tradeoff between processing time and storage costs. The decisions you'll need to make here are:

* Is it possible to re-process all data within RTO whether on primary or recovery site
* Is it cheaper to re-process, or replicate data and restore it?

The answer to these questions will vary based on the unique requirements of your data set. There are several stages of data processing in most data platforms, and the manner in which data arrives (batch, realtime) as well as the frequency and size of that data will affect this decision. If you have large quantities of data arriving daily the decision may be different to small quantities trickling in. similarly, highly processed data may have a different solution compared to data with minimal transformation requirements.

To determine the best solution, pick a timespan to calculate over. Choose this based on how long the data will be useful. Next, calculate how much it will cost to re-produce that data and how much it will cost to store that data for the timespan it will be useful. Also, record how long it takes to re-process the data. These calculations will give you a good indicator of which is the best solution.

**Raw Data** For raw data, you need to balance whether you can ingest into the recovery site in a reasonable time without adversely impacting systems of record (Production data systems)

**Curated Data** For curated data this decision might be based on whether the processing produces multiple files or replaces a single file. Where new files are created, re-processing would be less since only a small data set needs to be processed to recover data. Where a single, larger file is output and replaces older files recovery of the file is less useful since it would be overwritten anyway.

## Decoupled Infrastructure

In cloud scenarios is it often useful to decouple infrastructure to allow for underlying systems to change without complex reconfiguration. This is most often done using global load balancers in analytical data solutions. Real time solutions such as IoT and integration platforms may also achieve this using service bus technology for guaranteed delivery of data.
![globalloadbalance.png](images/globalloadbalance.png)

## Regional Service Availability

When working with geo-redundancy you must ensure that all services are available in the target region. Since not all services are available in every region, you may find that your data is available but the services to consume it are not in the partner region. This would be a disaster in and of itself and so must be avoided to ensure recovery is possible. Use the [Service Availability by Region](https://azure.microsoft.com/en-gb/global-infrastructure/services/) page to check availability of your services.