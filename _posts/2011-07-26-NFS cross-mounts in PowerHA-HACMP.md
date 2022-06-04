---
title: "NFS cross-mounts in PowerHA"
date: "2011-07-26"
categories: 
  - "aix"
  - "powerha"
tags: 
  - "aix"
  - "nfs"
  - "powerha"
---

Combining NFS with PowerHA we can achieve a HANFS (Highly Available Network File System). The basic concept behind this solution is that one node in the cluster mounts the resource locally, and offers that as an exported resource via a serviceable IP. Another node in the cluster is then configured to take on the resource in the event of failure.

If you're following this, I'm taking the assumption that your cluster is already configured, you have a working IP network and have set up a shared volume group between the cluster nodes that will be handling the HANFS failover. Before we get started though, there are a few things which need to be installed/verified.

## Prerequisites

The `cluster.es.nfs.rte` fileset needs to be installed so that PowerHA can work with HANFS.

```console
# lslpp -l cluster.es.nfs.rte
  Fileset                      Level  State      Description
  ----------------------------------------------------------------------------
Path: /usr/lib/objrepos
  cluster.es.nfs.rte         5.5.0.1  COMMITTED  ES NFS Support
 
Path: /etc/objrepos
  cluster.es.nfs.rte         5.5.0.1  COMMITTED  ES NFS Support
```

Also, since we're dealing with NFS, we need to make sure we have the portmapper daemon running.

```console
# lssrc -s portmap
Subsystem         Group            PID          Status
 portmap          portmap          213160       active
```

To have the portmapper daemon start during system boot, you'll need to modify `/etc/rc.tcpip` and uncomment the start line below.

```plaintext
# Start up Portmapper
start /usr/sbin/portmap "$src_running"
```
{: file="/etc/rc.tcpip" }

## HANFS layout

Below is a table of what the HANFS layout looks like.

| Resource Group | Application Server | Service IP Label | Shared Volume Group | Shared Logical Volumes | Shared Filesystems   | Mount Point    |
| -------------- | ------------------ | ---------------- | ------------------- | ---------------------- | -------------------- | -------------- |
| RG_nfs         | app_nfsv4          | serviceip-nfs    | haNFSvg             | hanfs_kris_lv          | /hanfs_home_kristian | /home/kristian |
|                |                    |                  |                     | hanfs_data_lv          | /hanfs_data          | /data          |
|                |                    |                  |                     | hanfs_nfsstable_lv     | /nfs_stable          | /nfs_stable    |

We have a Resource Group (RG\_nfs) which contains an Application Server (app\_nfsv4) and a Service IP Label (serviceip-nfs). There is a Shared Volume Group (haNFSvg) which contains some Shared Logical Volumes & Filesystems. The thing to take note here, is that we are only exporting two of the three filesystems that you see in the table. The /nfs\_stable filesystem will be the NFSv4 stable storage path location.

> "Stable Storage is a file system space that is used to save the state information by the NFSv4 server. This is very crucial for maintaining NFSv4 client's state information to facilitate smooth and transparent fallover/fallback/move of the Resource group from one node to other."

## Configuration

With the above information in hand, we're ready to start configuring HANFS.

1. (All Nodes) Set the NFS domain

    The NFSv4 domain needs to be set on all cluster nodes which will be responsible for taking on the RG_nfs resource group in the event of a failure.

    ```console
    # chnfsdom                  <- This will show the current NFS domain
    # chnfsdom [new_domain]     <- To set a new NFS domain
    ```

2. (All Nodes) Create mount points

    The mount points need to be created on all cluster nodes which will be responsible for taking on the RG_nfs resource group in the event of a failure.

    ```console
    # mkdir -p /home/kristian
    # mkdir -p /data
    ```

    Take note that we're only creating the mount points here, and not the logical volumes or filesystems.

3. Create logical volumes and filesystems

    Now we create the following logical volumes:

    - `hanfs_kris_lv`
    - `hanfs_data_lv`
    - `hanfs_nfsstable_lv`

    Create the logical volumes using the following path.

    ```console
    # smit hacmp
        -> System Management (C-SPOC)
          -> HACMP Logical Volume Management
            -> Shared Logical Volumes
              -> Add a Shared Logical Volume
    ```

    Create the filesystems using the following path.

    ```console
    # smit hacmp
        -> System Management (C-SPOC)
          -> HACMP Logical Volume Management
            -> Shared File Systems
              -> Enhanced Journaled File Systems
                -> Add an Enhanced Journaled File System on a Previously Defined Logical Volume
    ```

    We should now see the 2 filesystems which we are exporting and the stable storage filesystem apart of the shared volume group

    ```console
    # lsvg -l haNFSvg
    haNFSvg:
    LV NAME              TYPE       LPs     PPs     PVs  LV STATE       MOUNT POINT
    hanfs_kris_lv        jfs2       107     107     1    open/syncd     /hanfs_home_kristian
    hanfs_data_lv        jfs2       32      32      1    open/syncd     /hanfs_data
    hanfs_nfsstable_lv   jfs2       75      75      1    open/syncd     /nfs_stable
    ```

4. Configure the Application Server

    Luckily for us, IBM have made this step rather easy and have provided start, stop and monitor scripts. The location of these scripts are below.

    - `/usr/es/sbin/cluster/apps/clas_nfsv4/start`
    - `/usr/es/sbin/cluster/apps/clas_nfsv4/stop`
    - `/usr/es/sbin/cluster/apps/clam_nfsv4/monitor`

    Create an Application Server using the following path.

    ```console
    # smit hacmp
        -> Extended Configuration
          -> Extended Resource Configuration
            -> HACMP Extended Resources Configuration
              -> Configure HACMP Applications Servers
                -> Add an Application Server
    ```

    To configure Application Server monitoring, use the following path.

    ```console
    # smit hacmp
        -> Extended Configuration
          -> Extended Resource Configuration
            -> HACMP Extended Resources Configuration
              -> Configure HACMP Application Monitoring
                -> Configure Custom Application Monitors
                  -> Add a Custom Application Monitor
    ```

5. Configure the Resource Group for HANFS

    The next step is to configure the RG_nfs resource group with the values needed for HANFS.

    Modify the resource group using the following path.

    ```console
    # smit hacmp
        -> Extended Configuration
          -> Extended Resource Configuration
            -> HACMP Extended Resource Group Configuration
              -> Change/Show Resources and Attributes for a Resource Group
    ```

    ```console
    Service IP Labels/Addresses                     [serviceip-nfs]
    Application Servers                             [app_nfsv4]
    
    Filesystems (empty is ALL for VGs specified)    [/hanfs_home_kristian /hanfs_data /nfs_stable]
    Filesystems Consistency Check                   fsck
    Filesystems Recovery Method                     sequential
    Filesystems mounted before IP configured        true
    Filesystems/Directories to Export (NFSv2/3)     []

    Filesystems/Directories to Export (NFSv4)       [/hanfs_home_kristian /hanfs_data]
    Stable Storage Path (NFSv4)                     [/nfs_stable]
    Filesystems/Directories to NFS Mount            [/home/kristian;/hanfs_home_kristian /data;/hanfs_data]
    ```

    Most vaules are self explainatory. We set "Filesystems mounted before IP configured" to true so we prevent access from clients before the filesystems are ready. We also specify mount points in the following format `[mount point];[exported filesystem]`

6. HANFS exports file

    Just like NFS has `/etc/exports`, HANFS has `/usr/es/sbin/cluster/etc/exports`. If you need to specify NFS options, you MUST use `/usr/es/sbin/cluster/etc/exports` and not `/etc/exports`. For help creating the exports file, you can use `smit mknfsexp`.

    ```console
    * Pathname of directory to export                   []
    Anonymous UID                                     [-2]
    Public filesystem?                                no
    * Export directory now, system restart or both      both
    Pathname of alternate exports file                [/usr/es/sbin/cluster/etc/exports]
    ...
    ...
    ```

7. Synchronize the cluster

    We now need to synchronize our changes to the other cluster nodes

    ```console
    # smit hacmp
        -> Extended Configuration
          -> Extended Verification and Synchronization
    ```

8. Bring the Resource Group online

    We now bring the resource group online. It's a good idea at this stage to tail the hacmp.out file to see any errors.

    To bring the resource group online.

    ```console
    # smit hacmp
        ->  System Management (C-SPOC)
          -> HACMP Resource Group and Application Management
            -> Bring a Resource Group Online
    ```

    ```console
    # tail -f /var/hacmp/log/hacmp.out
    ...
    +RG_nfs:cl_activate_nfs(.110):/home/kristian;/hanfs_home_kristian[nfs_mount+102] : Attempt 0/5 to NFS-mount at Jul 26 11:01:21.000
    +RG_nfs:cl_activate_nfs(.110):/home/kristian;/hanfs_home_kristian[nfs_mount+103] mount -o vers=4,hard,intr serviceip-nfs:/hanfs_home_kristian /home/kristian
    ```

    You should now see the filesystems mounted.

    ```console
    # mount
    node          mounted                     mounted over            vfs         date            options
    --------        --------------              ---------------         ------      ------------    ---------------
    ...
                    /dev/hanfs_kris_lv          /hanfs_home_kristian    jfs2        Jul 26 14:36    rw,log=INLINE
                    /dev/hanfs_data_lv          /hanfs_data             jfs2        Jul 26 14:36    rw,log=INLINE
                    /dev/hanfs_nfsstable_lv     /nfs_stable             jfs2        Jul 26 14:36    rw,log=INLINE
    serviceip-nfs   /hanfs_home_kristian        /home/kristian          nfs4        Jul 26 14:36    vers=4,hard,intr
    serviceip-nfs   /hanfs_data                 /data                   nfs4        Jul 26 14:36    vers=4,hard,intr
    ```

    What you're seeing above is the output from the mount command ran from the cluster node which currently has the RG_nfs resource group ONLINE. You'll notice that it has the shared logical volumes and filesystems mounted, then a local NFS export from itself to mount `/home/kristian` and `/data`. On the other cluster nodes, you will only see the NFS mounts.

There you have it, NFS cross-mounts with PowerHA.
