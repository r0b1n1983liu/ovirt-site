---
title: Upgrading a Remote Database Environment from 4.1 to oVirt 4.2
---

# Chapter 7: Upgrading a Remote Database Environment from 4.1 to oVirt 4.2

## Planning an Upgrade from a 4.1 Environment with a Remote Database to 4.2

Upgrading your environment from 4.1 to 4.2 includes the following steps:

1. Upgrade the database from PostgreSQL 9.2 to 9.5

2. Update the 4.1 Engine to the latest version of 4.1

3. Upgrade the Engine from 4.1 to 4.2

4. Upgrade the hosts

5. Perform [post-upgrade tasks](chap-Post-Upgrade_Tasks/)

## Upgrading Remote Databases

oVirt 4.2 uses PostgreSQL 9.5 instead of PostgreSQL 9.2. If your databases are installed locally, the upgrade script will automatically upgrade them from version 9.2 to 9.5, and you can skip this section and proceed to the next step. However, if either of your databases (Engine or Data Warehouse) is installed on a separate machine, you must perform the following procedure on each remote database before upgrading the Engine.

1. Stop the service running on the machine:

  * Stop the ovirt-engine service on the Manager machine:

          # systemctl stop ovirt-engine

  * Stop the ovirt-engine-dwh service on the Data Warehouse machine:

          # systemctl stop ovirt-engine-dwhd

2. Install the PostgreSQL 9.5 packages:

        # yum install rh-postgresql95 rh-postgresql95-postgresql-contrib

3. Stop and disable the PostgreSQL 9.2 service:

        # systemctl stop postgresql
        # systemctl disable postgresql

4. Upgrade the PostgreSQL 9.2 database to PostgreSQL 9.5:

        # scl enable rh-postgresql95 -- postgresql-setup upgrade

5. Start and enable the `rh-postgresql95-postgresql.service` and check that it is running:

        # systemctl start rh-postgresql95-postgresql.service
        # systemctl enable rh-postgresql95-postgresql.service
        # systemctl status rh-postgresql95-postgresql.service

  Ensure that you see an output similar to the following:

        rh-postgresql95-postgresql.service - PostgreSQL database server
           Loaded: loaded (/usr/lib/systemd/system/rh-postgresql95-postgresql.service;
        enabled; vendor preset: disabled)
           Active: active (running) since Mon 2018-05-07 08:48:27 CEST; 1h 59min ago

6. Log in to the database and enable the uuid-ossp extension:

        # su - postgres -c "scl enable rh-postgresql95 -- psql -d database-name"

7. Execute the following SQL commands:

        # database-name=# DROP FUNCTION IF EXISTS uuid_generate_v1();
        # database-name=# CREATE EXTENSION "uuid-ossp";

8. Copy the `pg_hba.conf` client configuration file from the 9.2 environment to your 9.5 environment:

        # cp -p /var/lib/pgsql/data/pg_hba.conf  /var/opt/rh/rh-postgresql95/lib/pgsql/data/pg_hba.conf

9. Update the following parameters in the postgresql.conf file:

# vi /var/opt/rh/rh-postgresql95/lib/pgsql/data/postgresql.conf

        listen_addresses='\*'
        autovacuum_vacuum_scale_factor='0.01'
        autovacuum_analyze_scale_factor='0.075'
        autovacuum_max_workers='6'
        maintenance_work_mem='65536'
        max_connections='150'
        work_mem = '8192'

10. Restart the PostgreSQL 9.5 service to apply the configuration changes:

        # systemctl restart rh-postgresql95-postgresql.service

The remote databases have been upgraded.

## Update the oVirt Engine

Updates to the oVirt Engine are released through the oVirt repositories.

**Procedure**

1. On the oVirt Engine machine, check if updated packages are available:

        # engine-upgrade-check

    **Note:** If updates are expected, but not available, enable the required repositories. See Enabling the oVirt Engine Repositories in the Installation Guide.

2. Update the setup packages:

    # yum update ovirt\*setup\*

    # yum update rhevm-setup

3. Update the oVirt Engine. The `engine-setup` script prompts you with some configuration questions, then stops the **ovirt-engine** service, downloads and installs the updated packages, backs up and updates the database, performs post-installation configuration, and starts the **ovirt-engine** service.

        # engine-setup

    **Note:** The `engine-setup` script is also used during the oVirt Engine installation process, and it stores the configuration values supplied. During an update, the stored values are displayed when previewing the configuration, and may not be up to date if `engine-config` was used to update configuration after installation. For example, if `engine-config` was used to update `SANWipeAfterDelete` to `true` after installation, `engine-setup` will output "Default SAN wipe after delete: False" in the configuration preview. However, the updated values will not be overwritten by `engine-setup`.

    **Important:** The update process may take some time; allow time for the update process to complete and do not stop the process once initiated.

4. Update the base operating system and any optional packages installed on the Engine:

        # yum update

    **Important:** If the update upgraded any kernel packages, reboot the system to complete the changes.

## Upgrading the Engine from 4.1 to 4.2

**Prerequisites**

This procedure assumes that the system on which the Manager is installed is attached to the subscriptions for receiving oVirt 4.1 packages.

    **Important:** If the upgrade fails, the `engine-setup` command will attempt to roll your oVirt Engine installation back to its previous state. For this reason, the repositories required by oVirt 4.0 must not be removed until after the upgrade is complete. If the upgrade fails, detailed instructions display that explain how to restore your installation.

**Procedure**

1. Update the setup packages:

        # yum update ovirt\*setup\*

2. Run the following command and follow the prompts to upgrade the oVirt Engine:

        # engine-setup

3. Remove or disable the oVirt Engine 4.1 repositories to ensure the system does not use any oVirt Engine 4.1 packages:

        # subscription-manager repos --disable=rhel-7-server-rhv-4.1-rpms
        # subscription-manager repos --disable=rhel-7-server-rhv-4.1-manager-rpms
        # subscription-manager repos --disable=rhel-7-server-rhv-4-tools-rpms

4. Update the base operating system:

        # yum update

          **Important:** If any kernel packages were updated, reboot the system to complete the update.

You can now update the hosts.

## Update the Hosts

Use the host upgrade manager to update individual hosts directly from the oVirt Engine.

oVirt 3.6 supported three types of host. If you are using Enterprise Linux hosts or oVirt Nodes 3.6, use this procedure to update them. However, if you are using oVirt Node 4.0+, you must reinstall them with oVirt Node 4.2.

If you are not sure if you are using oVirt Node 3.6 or oVirt Node 4.0+, run:

    # imgbase check

If the command fails, the host is oVirt Node 3.6. If the command succeeds, the host is oVirt Node.

    **Note:** The upgrade manager only checks host with a status of `Up` or `Non-operational`, but not `Maintenance`.

    **Important:** On oVirt Node 4.0+, the update only preserves modified content in the /etc and /var directories. Modified data in other paths is overwritten during an update.

**Prerequisites**

* If migration is enabled at cluster level, virtual machines are automatically migrated to another host in the cluster; as a result, it is recommended that host updates are performed at a time when the host’s usage is relatively low.

* Ensure that the cluster contains more than one host before performing an update. Do not attempt to update all the hosts at the same time, as one host must remain available to perform Storage Pool Manager (SPM) tasks.

* Ensure that the cluster to which the host belongs has sufficient memory reserve in order for its hosts to perform maintenance. If a cluster lacks sufficient memory, the virtual machine migration operation will hang and then fail. You can reduce the memory usage of this operation by shutting down some or all virtual machines before updating the host.
* You cannot migrate a virtual machine using a vGPU to a different host. Virtual machines with vGPUs installed must be shut down before updating the host.

**Procedure**

1. If your Enterprise Linux hosts are set to version 7.3, you must reset them to the general EL 7 version before updating.

        # subscription-manager release --set=7Server

2. Disable your current repositories:

        # subscription-manager repos --disable=*

3. Ensure the correct repositories are enabled. You can check which repositories are currently enabled by running yum repolist.

  * For oVirt Nodes:

        # subscription-manager repos --enable=rhel-7-server-oVirt Node-4-rpms

  * For Enterprise Linux hosts:

        # subscription-manager repos --enable=rhel-7-server-rpms
        # subscription-manager repos --enable=rhel-7-server-rhv-4-mgmt-agent-rpms
        # subscription-manager repos --enable=rhel-7-server-ansible-2-rpms

4. Click **Compute** &rarr; **Hosts** and select the host to be updated.

5. Click **Installation** &rarr; **Check** for Upgrade.

6. Click **OK** to begin the upgrade check.

7. Click **Installation** &rarr; **Upgrade**.

8. Click **OK** to update the host. The details of the host are updated in Compute → Hosts and the status will transition through these stages:

  * **Maintenance**

  * **Installing**

  * **Up**

   After the update, the host is rebooted. Once successfully updated, the host displays a status of `Up`. If any virtual machines were migrated off the host, they are now migrated back.

    **Note:** If the update fails, the host’s status changes to Install Failed. From Install Failed you can click **Installation** &rarr; **Upgrade** again.

Repeat this procedure for each host in the oVirt environment.

**Prev:** [Chapter 6: Upgrading a Remote Database Environment from 4.0 to oVirt 4.2](chap-Upgrading_a_Remote_Database_Environment_from_4.0_to_oVirt_4.2)<br>
**Next:** [Chapter 8: Post-Upgrade Tasks](chap-Post-Upgrade_Tasks/)

[Adapted from RHV 4.2 documentation - CC-BY-SA](https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.2/html/upgrade_guide/chap-remote_upgrading_to_red_hat_virtualization_4.2)
