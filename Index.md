# Backup and DR for Data Platforms
**Dave Lusty**

# Contents

* [Introduction](#Introduction)
  * [Cloud Compute](#CloudCompute)
  * [Archive](#IntroductionArchive)
  * [Disaster Recovery](#IntroductionDisasterRecovery)
  * [Service Availability](#IntroductionServiceAvailability)
* [General Requirements](#GeneralRequirements)
  * [Systems Of Record](#SystemsOfRecord)
    * [Backup](#SystemsOfRecordBackup)
    * [Disaster Recovery](#SystemsOfRecordDisasterRecovery)
  * [Ingest](#Ingest)
    * [Backup](#IngestBackup)
    * [Disaster Recovery](#IngestDisasterRecovery)
  * [Store](#Store)
    * [Backup](#StoreBackup)
    * [Disaster Recovery](#StoreDisasterRecovery)
  * [Prep and Train](#PrepAndTrain)
    * [Backup](#PrepAndTrainBackup)
    * [Disaster Recovery](#PrepAndTrainDisasterRecovery)
  * [Model](#Model)
    * [Backup](#ModelBackup)
    * [Disaster Recovery](#ModelDisasterRecovery)
  * [Serve](#Serve)
    * [Backup](#ServeBackup)
    * [Disaster Recovery](#ServeDisasterRecovery)
* [SQL](#SQL)
  * [SQL Managed Instance](#SQLSQLManagedInstance)
  * [SQL Server](#SQLSQLServer)
  * [Azure SQL Database](#SQLAzureSQLDatabase)
* [Data Factory](#DataFactory)
  * [Integration Runtime](#DataFactoryIntegrationRuntime)
  * [Pipelines](#DataFactoryPipelines)
* [Data Lake](#DataLake)
  * [Storage Account (blob)](#DataLakeStorageAccount)
    * [Blob Lifecycle Management](#DataLakeStorageAccountBlobLifecycleManagement)
  * [Azure Data Lake Store Gen 2](#DataLakeStorageAccountAzureDataLakeStoreGen2)
  * [Azure Data Lake Store Gen 1](#DataLakeStorageAccountAzureDataLakeStoreGen1)
* [HDInsight](#HDInsight)
* [Azure Databricks](#AzureDatabricks)
* [SQL Data Warehouse](#SQLDataWarehouse)
* [Azure Analysis Services](#AzureAnalysisServices)
* [Power BI](#PowerBI)


# <a name="Introduction"></a>Introduction
This guide aims to show effective ways of backing up and recovering a cloud data platform, as well as how to recover in a regional outage or disaster recovery situation. In this instance cloud data platform refers specifically to cloud data warehousing using data lake and Data Factory such as in the image below.
![Data Platform Overview](images/DataMarts.png)

## <a name="CloudCompute"></a>Cloud Compute
Understanding how to create a suitable strategy in the cloud requires an understanding of cloud compute itself. Your strategy will be a mixture of requirements and cost/performance. In the cloud, compute tends to be expensive while storage is cheap. As such it may often be more suitable to store multiple full and complete copies of data rather than incremental backup, or recreating data. For instance, with a data warehouse it might be that you choose not to back it up at all. Instead you may create a complete new set of tables in your storage every day and do a complete load daily. With this setup, a recovery would be as simple as reloading from a given set of files. Since storage is relatively cheap there is no problem keeping a month of daily versions of the full dataset in most instances.

## <a name="IntroductionArchive"></a>Archive
Data platforms are often used for longer term retention of information which may have been removed from systems of record. For instance, consolidated sales information may be kept long term in a data platform even when the original data is removed. In this scenario, treat the archive data as a system in its own right and not as a backup of source data. This means you should take backups or snapshots of the archive data, as well as potentially replicating the archive to a recovery site.
Backup and Restore
Cloud platform backup and recovery is often a big change to traditional off-site tape backup. Firstly, we need to be clear of the purposes of the backup. The traditional tape solution was used for everything since it was the only avenue to any recovery. As such it included recovering data, recovering systems, and recovering during a DR scenario. In the cloud the connected nature of the platform, and the on-demand provisioning, change this viewpoint. Since we are now able to provision on demand it is generally preferable to store the deployment configuration rather than a full system image. Since we have multiple, well connected regions it is preferable to have a cold, warm, or hot standby site rather than recover from scratch in a disaster. As such, backup may be seen as primarily a way to recover data following corruption or accidental modification. Scenarios to consider for backup are:
* Accidental data modification
* Data corruption
* Accidental configuration change
Retention of backups as a form of archival should be avoided unless specifically called for in regulation. As a rule, backup copies do not aide compliance and in cases such as GDPR will often harm compliance. Archival of data will not be covered in this document and is considered a separate subject.
## <a name="IntroductionDisasterRecovery"></a>Disaster Recovery
In cloud solutions, disaster recovery is very different to on-premises. Here, considerations such as speed of deployment and cost may affect the decision as to how you will recover. More than ever, the RPO and RTO requirements must be realistic and specific to the scenario. Where data is available in a replicated storage account, the difference between live continuity using clustering, booting a cold system, and building a new system from scratch might be around 30 minutes but many thousands of dollars in consumption. It is imperative, therefore to determine whether that 30 minute windows while a database server deploys are worth the cost of leaving such a system running. In data platforms especially it is likely the workload is batch and would need to re-start a processing job anyway. Additionally, the likelihood of a regional outage from a global cloud provider is lower than that of on-premises solutions. The security and change controls alone make accidental outages less likely while advanced facility design and provisioning ensure that failure is less likely and may be mitigated rather than allowing a failure. Design of the platform ensures that many components within region are resilient or can be swapped out more quickly than a full DR failover. Again, the RTO comes into play as we need to balance the speed and complexity of a failover with the speed and efficiency of the cloud provider recovering. Data should always be made available in a secondary region, or saved to a location off-cloud. In the case of data platforms, much of the data is build from systems of record anyway and so can potentially be recreated rather than restored. Again, cost will come into the equation as you need to determine whether it’s cheaper to recreate and reprocess data or to recover it after storing backup copies.
## <a name="IntroductionServiceAvailability"></a>Service Availability
When working with geo-redundancy you must ensure that all services are available in the target region. Since not all services are available in every region, you may find that your data is available but the services to consume it are not in the partner region.
# <a name="GeneralRequirements"></a>General Requirements
This section will discuss whether or not backups are required for various components in a very generic way. This is intended to help you form a backup and recovery strategy for your own data that can then be translated to the services your data platform is actually using, whether Microsoft or third party. The common Microsoft components are detailed in later sections for implementation specifics.
![Data Platform Overview](images/DataMarts.png)
## <a name="SystemsOfRecord"></a>Systems Of Record
### <a name="SystemsOfRecordBackup"></a>Backup
These data sources, such as existing SQL Databases, CRM, ERP and other systems should already have backups in place. They are generally line of business and the data is therefore neither transient nor trivial.
### <a name="SystemsOfRecordDisasterRecovery"></a>Disaster Recovery
A disaster recovery plan should already be in place for these data sources where appropriate.
## <a name="Ingest"></a>Ingest
### <a name="IngestBackup"></a>Backup
Generally, the ingest stage is transient and therefore there will not be any data to back up here. The scripts and configuration for this component should be backed up, however. As changes are made to scripts it is possible for configuration errors to be introduced which may need to be backed off. Since the ingest of data will change the data in the lake, longer term retention may not be required. Ideally a DevOps style approach using code management such as GitHub or an Azure DevOps code repository should be used to maintain code versioning and to deploy new versions in a controlled way. Alternatively change control can be used in more manual processes.
### <a name="IngestDisasterRecovery"></a>Disaster Recovery
DR for ingest services can be complex and is split into two sides. There are on premises components such as data gateways which need to be protected in line with your other on premises systems. This protects against an outage of your on premises data centre. Secondly is the protection of the cloud components. These need to maintain connectivity to data sources from the recovery data centre or region. Since there is no data within the service, it is often simpler to redeploy and configure in DR than to maintain a primary and secondary instance. 
## <a name="Store"></a>Store
![Data Staging](images/DataStaging.png)

There are usually various stages of data within the data lake such as raw, curated, enriched etc. and each of these may need to be treated separately based on business value and how long they might take to recreate. The definitions of these may also vary based on use-case. For instance raw data in an IoT scenario might be kept long term as a kind of “system of record” while in a batch system the raw incoming data is only needed until a curated copy is made and backed up.
The questions to ask here are:
* Can I recreate the data from source later?
* Will the source retain the data as long as I plan to keep this copy?
* Will it cost less to back up this data than to re-create it?
* Will it take longer to re-create than my RTO allows?

### <a name="StoreBackup"></a>Backup
### <a name="StoreDisasterRecovery"></a>Disaster Recovery
## <a name="PrepAndTrain"></a>Prep and Train
### <a name="PrepAndTrainBackup"></a>Backup
### <a name="PrepAndTrainDisasterRecovery"></a>Disaster Recovery
## <a name="Model"></a>Model
### <a name="ModelBackup"></a>Backup
### <a name="ModelDisasterRecovery"></a>Disaster Recovery
## <a name="Serve"></a>Serve
### <a name="ServeBackup"></a>Backup
### <a name="ServeDisasterRecovery"></a>Disaster Recovery
# <a name="SQL"></a>SQL
## <a name="SQLSQLManagedInstance"></a>SQL Managed Instance
## <a name="SQLSQLServer"></a>SQL Server
## <a name="SQLAzureSQLDatabase"></a>Azure SQL Database
# <a name="DataFactory"></a>Data Factory
Config in Github
## <a name="DataFactoryIntegrationRuntime"></a>Integration Runtime
Second copy
## <a name="DataFactoryPipelines"></a>Pipelines
# <a name="DataLake"></a>Data Lake
## <a name="DataLakeStorageAccount"></a>Storage Account (blob)
[https://azure.microsoft.com/en-us/services/storage/blobs/](https://azure.microsoft.com/en-us/services/storage/blobs/)

Hot, cool, archive
To read data in Archive storage, you must first change the tier of the blob to Hot or Cool. This process is known as rehydration and can take up to 15 hours to complete. Large blob sizes are recommended for optimal performance. Rehydrating several small blobs concurrently may add additional time.

### <a name="DataLakeStorageAccountBlobLifecycleManagement"></a>Blob Lifecycle Management
Blob Storage lifecycle management (Preview) offers a rich, rule-based policy that you can use to transition your data to the best access tier and to expire data at the end of its lifecycle. See Manage the Azure Blob storage lifecycle to learn more. 

Snapshots are at the blob level so probably best to integrate into Azure Data Factory pipelines

## <a name="DataLakeStorageAccountAzureDataLakeStoreGen2"></a>Azure Data Lake Store Gen 2
No soft delete or snapshots
## <a name="DataLakeStorageAccountAzureDataLakeStoreGen1"></a>Azure Data Lake Store Gen 1
Does not have GRS!
No snapshots
Must copy to Blob for backup capability
# <a name="HDInsight"></a>HDInsight
“stateless”
Scripted creation – Git versioning
Data lives on lake
# <a name="AzureDatabricks"></a>Azure Databricks
“stateless”
Scripted creation – Git versioning
Data lives on lake
Git integration for notebooks
# <a name="SQLDataWarehouse"></a>SQL Data Warehouse
[https://docs.microsoft.com/en-us/azure/sql-data-warehouse/backup-and-restore](https://docs.microsoft.com/en-us/azure/sql-data-warehouse/backup-and-restore)

auto snapshots
user snapshots
geo replication

# <a name="AzureAnalysisServices"></a>Azure Analysis Services
Backup to geo storage and restore
# <a name="PowerBI"></a>Power BI
Backup files
Git for version control
redeploy
