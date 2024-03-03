---
title: Rename a Proxmox node
weight: 1
---
# Rename a Proxmox node

## Introduction
Changing the hostname of a conventional Linux host is easy: Change the name in `/etc/hosts` and `/etc/hostname`, reboot, and you're pretty much done. If the host is a Proxmox node, the process is a little more complicated: The node name is reflected in a handful of directories and configuration files, all of which must be updated to avoid negative consequences.

There's more: If the node is in a Proxmox cluster, you'll also need to contend with Corosync and `pmxcfs` or risk bringing your cluster down.

This page has instructions for both cases — [renaming a standalone node](#rename-a-standalone-node) and [renaming a node in a cluster](#rename-a-node-in-a-cluster).

## Instructions

### Rename a standalone node
1. Access the node's console via SSH or the Proxmox web GUI.

2. If needed, log in as root:
    ```shell
    $ sudo su
    ```

3. Update the hostname in `/etc/hosts` and `/etc/hostname`.

4. Create a new directory for time series data related to the node:
    ```shell
    $ mkdir /var/lib/rrdcached/db/pve2-node/<new-name>
    $ cp -p /var/lib/rrdcached/db/pve2-node/<old-name> /var/lib/rrdcached/db/pve2-node/<new-name>
    ```

5. Create a new directory for time series data related to the node's storage:
    ```shell
    $ mkdir /var/lib/rrdcached/db/pve2-storage/<new-name>
    $ cp -p /var/lib/rrdcached/db/pve2-storage/<old-name>/* /var/lib/rrdcached/db/pve2-storage/<new-name>
    ```

7. Create a new directory for `qemu-server` config files:
    ```shell
    $ mkdir -p /etc/pve/nodes/<new-name>/qemu-server
    $ mv /etc/pve/nodes/<old-name>/qemu-server/* /etc/pve/nodes/<new-name>/qemu-server
    ```

8. Remove the directories that reflect the old node name:
    ```shell
    $ rm -Rf /etc/pve/nodes/<old-name>
    $ rm -Rf /var/lib/rrdcached/db/pve2-node/<old-name>
    $ rm -Rf /var/lib/rrdcached/db/pve2-storage/<old-name>
    ```

9. Reboot the node:
    ```shell
    $ reboot
    ```

### Rename a node in a cluster
If your node is in an active cluster, you'll also have to stop Corosync, force `pmxcfs` into `local` mode, and update the Corosync configuration.

1. Follow steps 1–8 of [Rename a standalone node](#rename-a-standalone-node).

2. Stop the cluster and [force local mode](https://pve.proxmox.com/pve-docs/pmxcfs.8.html) so you can update the cluster configuration:
    ```shell
    $ systemctl stop pve-cluster
    $ systemctl stop corosync
    $ pmxcfs -l
    ```

3. Create a copy of `corosync.conf`:
    ```shell
    $ cp /etc/pve/corosync.conf /etc/pve/corosync.conf.new
    ```

4. Update the following in `/etc/pve/corosync.conf.new`:
	* The node's name
    * The `config_version`, incrementing it by 1

5. Create another copy of `corosync.conf` as a backup:
    ```shell
    $ cp /etc/pve/corosync.conf /etc/pve/corosync.conf.bak
    ```

6. Restart the cluster:
    ```shell
    $ killall pmxcfs
    $ systemctl start pve-cluster
    ```