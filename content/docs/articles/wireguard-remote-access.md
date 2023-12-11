---
title: Access resources on a remote server with Wireguard
weight: 1
---
# Access resources on a remote server with Wireguard

## Introduction
I self-host a few apps on my home network, using a small single-board computer as my server.

Until recently, I was only able to access my self-hosted apps at home, on my home network. That limitation was inconvenient, to say the least.

For example, one of the apps I self-host is [Actual](https://actualbudget.org/), an open-source budgeting app that totally changed the way I manage my personal finances. I love Actual, and I love the idea of keeping my financial data safe on my home network away from prying eyes. However, the inability to access Actual remotely meant I couldn't update my budget or add transaction data without being at home. Annoying!

My solution? Use [Wireguard](https://www.wireguard.com/) to create a secure VPN tunnel to my home server, allowing me to access my self-hosted apps remotely, whether I'm across the street or across the world.

## Use case

- I only care about accessing resources on a single server. I don't need remote access to anything else on my home network.

- I don't need to send all my internet traffic through the VPN tunnel, just the traffic to and from my server.

- My home server is running [Ubuntu 20.04.6 LTS](https://releases.ubuntu.com/focal/) on the ARMv7 architecture.

## Set up the server

### Install Wireguard on the server

### Generate the server keys

### Set up the server interface

## Set up the client

### Generate the client keys

### Set up the client interface

### Add the client to the server configuration