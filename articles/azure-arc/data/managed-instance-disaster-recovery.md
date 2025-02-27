---
title: Disaster recovery - Azure Arc-enabled SQL Managed Instance
description: Describes disaster recovery for Azure Arc-enabled SQL Managed Instance
services: azure-arc
ms.service: azure-arc
ms.subservice: azure-arc-data-sqlmi
ms.custom: event-tier1-build-2022
author: dnethi
ms.author: dinethi
ms.reviewer: mikeray
ms.date: 04/04/2023
ms.topic: conceptual
---

# Azure Arc-enabled SQL Managed Instance - disaster recovery 

To configure disaster recovery in Azure Arc-enabled SQL Managed Instance, set up Azure failover groups.

## Background

Azure failover groups use the same distributed availability groups technology that is in SQL Server. Because Azure Arc-enabled SQL Managed Instance runs on Kubernetes, there's no Windows failover cluster involved.  For more information, see [Distributed availability groups](/sql/database-engine/availability-groups/windows/distributed-availability-groups).

> [!NOTE]
> - The Azure Arc-enabled SQL Managed Instance in both geo-primary and geo-secondary sites need to be identical in terms of their compute & capacity, as well as service tiers they are deployed in.
> - Distributed availability groups can be set up for either General Purpose or Business Critical service tiers. 

## Prerequisites

The following prerequisites must be met before setting up failover groups between two Azure Arc-enabled SQL managed instances:

- An Azure Arc data controller and an Arc enabled SQL managed instance provisioned at the primary site with `--license-type` as one of `BasePrice` or `LicenseIncluded`. 
- An Azure Arc data controller and an Arc enabled SQL managed instance provisioned at the secondary site with identical configuration as the primary in terms of:
  - CPU
  - Memory
  - Storage
  - Service tier
  - Collation
  - Other instance settings
- The instance at the secondary site requires `--license-type` as `DisasterRecovery`

> [!NOTE]
> - It is important to specify the `--license-type` **during** the Azure Arc-enabled SQL MI creation. This will allow the DR instance to be seeded from the primary instance in the primary data center. Updating this property post deployment will not have the same effect.

## Deployment process

To set up an Azure failover group between two Azure Arc-enabled SQL managed instances, complete the following steps:

1. Create custom resource for distributed availability group at the primary site
1. Create custom resource for distributed availability group at the secondary site
1. Copy the binary data from the mirroring certificates 
1. Set up the distributed availability group between the primary and secondary sites
 either in `sync` mode or `async` mode

The following image shows a properly configured distributed availability group:

![Diagram showing a properly configured distributed availability group](.\media\business-continuity\distributed-availability-group.png)

## Synchronization modes

Failover groups in Azure Arc data services support two synchronization modes - `sync` and `async`. The synchronization mode directly impacts how the data is synchronized between the Azure Arc-enabled SQL managed instances, and potentially the performance on the primary managed instance. 

If primary and secondary sites are within a few miles of each other, use `sync` mode. Otherwise use `async` mode to avoid any performance impact on the primary site. 

## Configure Azure failover group - direct mode

Follow the steps below if the Azure Arc data services  are deployed in `directly` connected mode. 

Once the prerequisites are met, run the below command to set up Azure failover group between the two Azure Arc-enabled SQL managed instances:

```azurecli
az sql instance-failover-group-arc create --name <name of failover group> --mi <primary SQL MI> --partner-mi <Partner MI> --resource-group <name of RG> --partner-resource-group <name of partner MI RG>
```

Example:

```azurecli
az sql instance-failover-group-arc create --name sql-fog --mi sql1 --partner-mi sql2 --resource-group rg-name --partner-resource-group rg-name
```

The above command:

1. Creates the required custom resources on both primary and secondary sites
1. Copies the mirroring certificates and configures the failover group between the instances 

## Configure Azure failover group - indirect mode

Follow the steps below if Azure Arc data services are deployed in `indirectly` connected mode. 

1. Provision the managed instance in the primary site.

   ```azurecli
   az sql mi-arc create --name <primaryinstance> --tier bc --replicas 3 --k8s-namespace <namespace> --use-k8s
   ```

2. Switch context to the secondary cluster by running ```kubectl config use-context <secondarycluster>``` and provision the managed instance in the secondary site that will be the disaster recovery instance. At this point, the system databases are not part of the contained availability group.

    > [!NOTE]
    > It is important to specify `--license-type DisasterRecovery` **during** the Azure Arc-enabled SQL MI creation. This will allow the DR instance to be seeded from the primary instance in the primary data center. Updating this property post deployment will not have the same effect.

   ```azurecli
   az sql mi-arc create --name <secondaryinstance> --tier bc --replicas 3 --license-type DisasterRecovery --k8s-namespace <namespace> --use-k8s
   ```

3. Mirroring certificates - The binary data inside the Mirroring Certificate property of the Azure Arc-enabled SQL MI is needed for the Instance Failover Group CR (Custom Resource) creation. 

    This can be achieved in a few ways:

    (a) If using `az` CLI, generate the mirroring certificate file first, and then point to that file while configuring the Instance Failover Group so the binary data is read from the file and copied over into the CR. The cert files are not needed after failover group creation. 

    (b) If using `kubectl`, directly copy and paste the binary data from the Azure Arc-enabled SQL MI CR into the yaml file that will be used to create the Instance Failover Group. 


    Using (a) above: 

    Create the mirroring certificate file for primary instance:
    ```azurecli
    az sql mi-arc get-mirroring-cert --name <primaryinstance> --cert-file </path/name>.pem​ --k8s-namespace <namespace> --use-k8s
    ```

    Example:
    ```azurecli
    az sql mi-arc get-mirroring-cert --name sqlprimary --cert-file $HOME/sqlcerts/sqlprimary.pem​ --k8s-namespace my-namespace --use-k8s
    ```

    Connect to the secondary cluster and create the mirroring certificate file for secondary instance:

    ```azurecli
    az sql mi-arc get-mirroring-cert --name <secondaryinstance> --cert-file </path/name>.pem --k8s-namespace <namespace> --use-k8s
    ```

    Example:

    ```azurecli
    az sql mi-arc get-mirroring-cert --name sqlsecondary --cert-file $HOME/sqlcerts/sqlsecondary.pem --k8s-namespace my-namespace --use-k8s
    ```

    Once the mirroring certificate files are created, copy the certificate from the secondary instance to a shared/local path on the primary instance cluster and vice-versa. 

4. Create the failover group resource on both sites. 


   > [!NOTE]
   > Ensure the SQL instances have different names for both primary and secondary sites, and the `shared-name` value should be identical on both sites.
   
    ```azurecli
    az sql instance-failover-group-arc create --shared-name <name of failover group> --name <name for primary failover group resource> --mi <local SQL managed instance name> --role primary --partner-mi <partner SQL managed instance name>  --partner-mirroring-url tcp://<secondary IP> --partner-mirroring-cert-file <secondary.pem> --k8s-namespace <namespace> --use-k8s
    ```

    Example:
    ```azurecli
    az sql instance-failover-group-arc create --shared-name myfog --name primarycr --mi sqlinstance1 --role primary --partner-mi sqlinstance2  --partner-mirroring-url tcp://10.20.5.20:970 --partner-mirroring-cert-file $HOME/sqlcerts/sqlinstance2.pem --k8s-namespace my-namespace --use-k8s
    ```

    On the secondary instance, run the following command to set up the failover group custom resource. The `--partner-mirroring-cert-file` in this case should point to a path that has the mirroring certificate file generated from the primary instance as described in 3(a) above.

    ```azurecli
    az sql instance-failover-group-arc create --shared-name <name of failover group> --name <name for secondary failover group resource> --mi <local SQL managed instance name> --role secondary --partner-mi <partner SQL managed instance name>  --partner-mirroring-url tcp://<primary IP> --partner-mirroring-cert-file <primary.pem> --k8s-namespace <namespace> --use-k8s
    ```

    Example:
    ```azurecli
    az sql instance-failover-group-arc create --shared-name myfog --name secondarycr --mi sqlinstance2 --role secondary --partner-mi sqlinstance1  --partner-mirroring-url tcp://10.10.5.20:970 --partner-mirroring-cert-file $HOME/sqlcerts/sqlinstance1.pem --k8s-namespace my-namespace --use-k8s
    ```

## Retrieve Azure failover group health state

Information about the failover group such as primary role, secondary role, and the current health status can be viewed on the custom resource on either primary or secondary site. 

Run the below command on primary and/or the secondary site to list the failover groups custom resource:

```azurecli
kubectl get fog -n <namespace>
```

Describe the custom resource to retrieve the failover group status, as follows:

```azurecli
kubectl describe fog <failover group cr name> -n <namespace>
```

## Failover group operations

Once the failover group is set up between the managed instances, different failover operations can be performed depending on the circumstances. 

Possible failover scenarios are:

- The Azure Arc-enabled SQL managed instances at both sites are in healthy state and a failover needs to be performed: 
    + perform a manual failover from primary to secondary without data loss by setting `role=secondary` on the primary SQL MI.
   
- Primary site is unhealthy/unreachable and a failover needs to be performed:
   
   + the primary Azure Arc-enabled SQL managed instance is down/unhealthy/unreachable
   + the secondary Azure Arc-enabled SQL managed instance needs to be force-promoted to primary with potential data loss 
   + when the original primary Azure Arc-enabled SQL managed instance comes back online, it will report as `Primary` role and unhealthy state and needs to be forced into a `secondary` role so it can join the failover group and data can be synchronized.
    

## Manual failover (without data loss)

Use `az sql instance-failover-group-arc update ...` command group to initiate a failover from primary to secondary. Any pending transactions on the geo-primary instance are replicated over to the geo-secondary instance before the failover. 

### Directly connected mode
Run the following command to initiate a manual failover, in `direct` connected mode using ARM APIs:

```azurecli
az sql instance-failover-group-arc update --name <shared name of failover group> --mi <primary Azure Arc-enabled SQL MI> --role secondary --resource-group <resource group>
```
Example:

```azurecli
az sql instance-failover-group-arc update --name myfog --mi sqlmi1 --role secondary --resource-group myresourcegroup
```
### Indirectly connected mode
Run the following command to initiate a manual failover, in `indirect` connected mode using kubernetes APIs:

```azurecli
az sql instance-failover-group-arc update --name <name of failover group resource> --role secondary --k8s-namespace <namespace> --use-k8s 
```

Example:

```azurecli
az sql instance-failover-group-arc update --name myfog --role secondary --k8s-namespace my-namespace --use-k8s 
```

## Forced failover with data loss

In the circumstance when the geo-primary instance becomes unavailable, the following commands can be run on the geo-secondary DR instance to promote to primary with a forced failover incurring potential data loss.

On the geo-secondary DR instance, run the following command to promote it to primary role, with data loss.

> [!NOTE]
> If the `--partner-sync-mode` was configured as `sync`, it needs to be reset to `async` when the secondary is promoted to primary. 

### Directly connected mode
```azurecli
az sql instance-failover-group-arc update --name <shared name of failover group> --mi <secondary Azure Arc-enabled SQL MI> --role force-primary-allow-data-loss --resource-group <resource group> --partner-sync-mode async
```
Example:

```azurecli
az sql instance-failover-group-arc update --name myfog --mi sqlmi2 --role force-primary-allow-data-loss --resource-group myresourcegroup --partner-sync-mode async
```

### Indirectly connected mode
```azurecli
az sql instance-failover-group-arc update --k8s-namespace my-namespace --name secondarycr --use-k8s --role force-primary-allow-data-loss --partner-sync-mode async
```

When the geo-primary Azure Arc-enabled SQL MI instance becomes available, run the below command to bring it into the failover group and synchronize the data:

### Directly connected mode
```azurecli
az sql instance-failover-group-arc update --name <shared name of failover group> --mi <old primary Azure Arc-enabled SQL MI> --role force-secondary --resource-group <resource group>
```

### Indirectly connected mode
```azurecli
az sql instance-failover-group-arc update --k8s-namespace my-namespace --name secondarycr --use-k8s --role force-secondary
```
Optionally, the `--partner-sync-mode` can be configured back to `sync` mode if desired. 

At this point, if you plan to continue running the production workload off of the secondary site, the `--license-type` needs to be updated to either `BasePrice` or `LicenseIncluded` to initiate billing for the vCores consumed.

## Next steps

[Overview: Azure Arc-enabled SQL Managed Instance business continuity](managed-instance-business-continuity-overview.md)