# Backup and DR for Data Platforms
**Dave Lusty**

# Contents

* [Introduction](#Introduction)
 * [Archive](#Archive)

# <a name="Introduction"></a>Introduction
This guide aims to show effective ways of backing up and recovering a cloud data platform, as well as how to recover in a regional outage or disaster recovery situation. In this instance cloud data platform refers specifically to cloud data warehousing using data lake and Data Factory such as in the image below.
![Data Platform Overview](images/DataMarts.png)

## <a name="Archive"></a>Archive
Data platforms are often used for longer term retention of information which may have been removed from systems of record. For instance, consolidated sales information may be kept long term in a data platform even when the original data is removed. In this scenario, treat the archive data as a system in its own right and not as a backup of source data. This means you should take backups or snapshots of the archive data, as well as potentially replicating the archive to a recovery site.
Backup and Restore
Cloud platform backup and recovery is often a big change to traditional off-site tape backup. Firstly, we need to be clear of the purposes of the backup. The traditional tape solution was used for everything since it was the only avenue to any recovery. As such it included recovering data, recovering systems, and recovering during a DR scenario. In the cloud the connected nature of the platform, and the on-demand provisioning, change this viewpoint. Since we are now able to provision on demand it is generally preferable to store the deployment configuration rather than a full system image. Since we have multiple, well connected regions it is preferable to have a cold, warm, or hot standby site rather than recover from scratch in a disaster. As such, backup may be seen as primarily a way to recover data following corruption or accidental modification. Scenarios to consider for backup are:
