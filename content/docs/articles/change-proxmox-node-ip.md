---
title: Change a Proxmox node's IP address
weight: 2
---
# Rename a Proxmox node

## Introduction
Similar to [renaming a Proxmox node](/docs/articles/rename-proxmox-node), changing the IP address of a node in a Proxmox cluster isn't as straightforward as changing the IP address of a conventional Linux host. The IP address must be updated on every node in the cluster

## Instructions
### Change node's IP address
1. Update the node's IP address in the following files:
  	* `/etc/network/interfaces`
  	* `/etc/hosts`
2. Stop the cluster and [force local mode](https://pve.proxmox.com/pve-docs/pmxcfs.8.html) so you can update the cluster configuration:
    ```
    systemctl stop pve-cluster
    systemctl stop corosync
    pmxcfs -l
    ```
3. In `/etc/pve/corosync.conf` update:
	* The node's IP address
	* The IP addresses of other nodes in the cluster (if they're changing)
	* The `config_version`, incrementing it by 1

4. Restart the cluster:
    ```
    killall pmxcfs
    systemctl start pve-cluster
    ```



### Change IP address on subsequent nodes
If you're changing the IP address of multiple nodes in a cluster, repeat the instructions in this section for every additional node in the cluster. The process to follow depends on if the cluster's quorum is still intact.

#### Intact quorum
Update the node's IP address in the following files:
* `/etc/network/interfaces`
* `/etc/hosts`


#### Non-intact quorum
Follow the instructions in [Change IP address on first node](#change-IP-address-on-first-node) to update the cluster configuration on the subsequent node.