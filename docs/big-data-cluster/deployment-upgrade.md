---
title: Upgrade to a new release
titleSuffix: SQL Server Big Data Clusters
description: Learn how to upgrade SQL Server Big Data Clusters to a new release.
author: MikeRayMSFT 
ms.author: mikeray
ms.reviewer: mihaelab
ms.date: 01/07/2020
ms.topic: conceptual
ms.prod: sql
ms.technology: big-data-cluster
---

# How to upgrade [!INCLUDE[big-data-clusters-2019](../includes/ssbigdataclusters-ss-nover.md)]

[!INCLUDE[tsql-appliesto-ssver15-xxxx-xxxx-xxx](../includes/tsql-appliesto-ssver15-xxxx-xxxx-xxx.md)]

The upgrade path depends on the current version of SQL Server Big Data Cluster (BDC). To upgrade from a supported release, including general distribution release (GDR), cumulative update (CU), or quick fix engineering (QFE) update, you can upgrade in place. In-place upgrade from a customer technology preview (CTP) or release candidate version of BDC is not supported. You need to remove and recreate the cluster. The following sections describe the steps for each scenario:

- [Upgrade from supported release](#upgrade-from-supported-release)
- [Update a BDC deployment from CTP or release candidate](#update-a-bdc-deployment-from-ctp-or-release-candidate)

>[!NOTE]
>The first supported release of Big Data Clusters is SQL Server 2019 GDR1.

## Upgrade release notes

Before you proceed, check the [upgrade release notes for known issues](release-notes-big-data-cluster.md#known-issues).

## Upgrade from supported release

This section explains how to upgrade a SQL Server BDC from a supported release to a newer supported release.

1. Back up SQL Server master instance.
2. Back up HDFS.

   ```
   azdata bdc hdfs cp --from-path <path> --to-path <path>
   ```
   
   For example: 

   ```
   azdata bdc hdfs cp --from-path hdfs://user/hive/warehouse/%%D --to-path ./%%D
   ```

3. Update `azdata`.

   Follow the instructions for installing `azdata`. 
   - [Windows installer](/deploy-install-azdata-installer.md#install-azdata-with-the-microsoft-windows-installer)
   - [Linux package manager](deploy-install-azdata-linux-package.md)

   >[!NOTE]
   >If `azdata` was installed with `pip` you need to manually remove it before installing with the Windows installer or the Linux package manager.

1. Update the Big Data Cluster.

   ```
   azdata bdc upgrade -n <clusterName> -t <imageTag> -r <containerRegistry>/<containerRepository>
   ```

   For example, the following script uses `2019-CU1-ubuntu-16.04` image tag:

   ```
   azdata bdc upgrade -n bdc -t 2019-CU1-ubuntu-16.04 -r mcr.microsoft.com/mssql/bdc
   ```

>[!NOTE]
>The latest image tags are available at [SQL Server 2019 Big Data Clusters release notes](release-notes-big-data-cluster.md).

>[!IMPORTANT]
>If you use a private repository to pre-pull the images for deploying or upgrading BDC, ensure that the current build images as well as >the target build images are in the private repository. This enables successful rollback, if necessary. Also, if you changed the >credentials of the private repository since the original deployment, update the corresponding secret in Kubernetes before you upgrade. >azdata does not support updating the credentials through AZDATA_PASSWORD and AZDATA_USERNAME environment variables. Update the secret >using [kubectl edit secrets](https://kubernetes.io/docs/concepts/configuration/secret/#editing-a-secret). Upgrading using different >private repositories for current and target builds is not supported.

### Increase the timeout for the upgrade

A timeout can occur if certain components are not upgraded in the allocated time. The following code shows what the failure might look like:

   ```
   >azdata.EXE bdc upgrade --name <mssql-cluster>
   Upgrading cluster to version 15.0.4003

   NOTE: Cluster upgrade can take a significant amount of time depending on
   configuration, network speed, and the number of nodes in the cluster.

   Upgrading Control Plane.
   Control plane upgrade failed. Failed to upgrade controller.
   ```

To increase the timeouts for an upgrade, edit the upgrade config map. To edit the upgrade config map:

Run the following command:

   ```bash
   kubectl edit configmap controller-upgrade-configmap
   ```

Edit the following fields:

   **controllerUpgradeTimeoutInMinutes** Designates the number of minutes to wait for the controller or controller db to finish upgrading. Default is 5. Update to at least 20.
   **totalUpgradeTimeoutInMinutes**: Designates the combines amount of time for both the controller and controller db to finish upgrading (controller + controllerdb upgrade).Default is 10. Update to at least 40.
   **componentUpgradeTimeoutInMinutes**: Designates the amount of time that each subsequent phase of the upgrade has to complete. Default is 30. Update to 45.

Save and exit.

## Update a BDC deployment from CTP or release candidate

In-place upgrade from a CTP or release candidate build of SQL Server Big Data Clusters is not supported. The following section explains how to manually remove and recreate the cluster.

### Backup and delete the old cluster

Currently, there is no in place upgrade for big data clusters, the only way to upgrade to a new release is to manually remove and recreate the cluster. Each release has a unique version of `azdata` that is not compatible with the previous version. Also, if an older cluster had to download a container image on a new node, the latest image might not be compatible with the older images on the cluster. The newer image is pulled if you are using the `latest` image tag for in the deployment configuration file for the container settings. By default, each release has a specific image tag corresponding to the SQl Server release version. To upgrade to the latest release, use the following steps:

1. Before deleting the old cluster, back up the data on the SQL Server master instance and on HDFS. For the SQL Server master instance, you can use [SQL Server backup and restore](data-ingestion-restore-database.md). For HDFS, you [can copy out the data with `curl`](data-ingestion-curl.md).

1. Delete the old cluster with the `azdata delete cluster` command.

   ```bash
    azdata bdc delete --name <old-cluster-name>
   ```

   > [!Important]
   > Use the version of `azdata` that matches your cluster. Do not delete an older cluster with the newer version of `azdata`.

   > [!Note]
   > Issuing a `azdata bdc delete` command will result in all objects created within the namespace identified with the big data cluster name to be deleted, but not the namespace itself. Namespace can be reused for subsequent deployments as long as it is empty and no other applications were created within.

1. Uninstall the old version of `azdata`.

   ```powershell
   pip3 uninstall -r https://azdatacli.blob.core.windows.net/python/azdata/2019-rc1/requirements.txt
   ```

1. Install the latest version of `azdata`. The following commands install `azdata` from the latest release:

   **Windows:**

   ```powershell
   pip3 install -r https://aka.ms/azdata
   ```

   **Linux:**

   ```bash
   pip3 install -r https://aka.ms/azdata --user
   ```

   > [!IMPORTANT]
   > For each release, the path to the `n-1` version of `azdata` changes. Even if you previously installed `azdata`, you must reinstall from the latest path before creating the new cluster.

### <a id="azdataversion"></a> Verify the azdata version

Before deploying a new big data cluster, verify that you are using the latest version of `azdata` with the `--version` parameter:

```bash
azdata --version
```

### Install the new release

After removing the previous big data cluster and installing the latest `azdata`, deploy the new big data cluster by using the current deployment instructions. For more information, see [How to deploy [!INCLUDE[big-data-clusters-2019](../includes/ssbigdataclusters-ss-nover.md)] on Kubernetes](deployment-guidance.md). Then, restore any required databases or files.

## Next steps

For more information about big data clusters, see [What are [!INCLUDE[big-data-clusters-2019](../includes/ssbigdataclusters-ss-nover.md)]](big-data-cluster-overview.md).
