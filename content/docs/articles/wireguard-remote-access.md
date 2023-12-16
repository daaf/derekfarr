---
title: Access a remote server via Wireguard
weight: 1
---
# Access a remote server via Wireguard

## Introduction
I self-host a few apps on my home network, using a small single-board computer as my server.

Until recently I was only able to access my self-hosted apps at home on my home network. That limitation caused me a few problems.

For example, one of the apps I self-host is [Actual](https://actualbudget.org/), an open-source budgeting app that totally changed the way I manage my personal finances. I love Actual, and I love the idea of keeping my financial data on my home network away from prying eyes. However, my inability to access Actual remotely meant I couldn't update my budget or add transactions without being at home. Annoying!

My solution? Use [Wireguard](https://www.wireguard.com/) to create a secure VPN tunnel to my home server, allowing me to access my self-hosted apps remotely.

## Use case
- I only care about accessing resources on a single server. I don't need remote access to anything else on my home network.

- I plan on using my mobile phone as the "client" to access my server.

- I don't need to send all my internet traffic through the VPN tunnel, just the traffic to and from my server.

- My server is running [Ubuntu 20.04.6 LTS](https://releases.ubuntu.com/focal/) on the ARMv7 architecture.

{{<hint warning>}}
**A note on terminology**

In this article, I use the terms _server_ and _client_ to describe the two endpoints of a Wireguard tunnel. I also use the Wireguard term _peer_ to refer to endpoints in the abstract.
{{</hint>}}

## Set up the server

### Install Wireguard on the server
Use the [installation method](https://www.wireguard.com/install/) of your choice to install Wireguard. I'm on Ubuntu, so I used the `apt` package manager.
```shell
$ sudo apt install wireguard
```
Now you should be able to use the [`wg`](https://man7.org/linux/man-pages/man8/wg.8.html) command.

### Generate the server keys
Wireguard relies on encrypted keys for authentication between peers. Each peer needs to have a private key and an associated public key that's derived from the private key. 

Conveniently, Wireguard comes with a few commands to create and store private and public keys. You can even pipe those commands together, like so:
```shell
$ wg genkey | tee privatekey | wg pubkey > publickey
```
Here's a breakdown of this command:
* `wg genkey`: Generates a private key
* `tee privatekey`: Saves the private key in a file called `privatekey` and writes the private key to stdout.
* `wg pubkey > publickey`: Generates a public key from the private key and saves the public key in a file called `publickey`

### Set up the server interface
Armed with our private and public keys, it's time to set up the first interface. I chose to start with the interface for my server.

1. Create an interface called `wg0` using the [`ip`](https://man7.org/linux/man-pages/man8/ip.8.html) command.
    ```shell
    $ ip link add dev wg0 type wireguard
    ```
2. Assign a block of IP addresses to use for the interface and its peers. In my case, I don't expect to have a ton of peers connecting to this interface, so I'm using a relatively small range of 15 IP addresses.
    ```shell
    $ ip address add dev wg0 192.168.2.1/28
    ```
3. Create a config file for the `wg0` interface at `/etc/wireguard/wg0.conf`. Add the following configuration:
    ```
    [Interface]
    Address = 192.168.2.1/28
    SaveConfig = true
    ListenPort = 51820
    PrivateKey = serverprivatekey
    ```
    Here's a breakdown of the configuration file:
    * `Address`: The IP address of the interface, which should be in the range defined in step 2.
    * `SaveConfig`: Whether the configuration should be saved after the interface is shut down.
    * `ListenPort`: The port the interface is listening on.
    * `PrivateKey`: The private key you generated in [_Generate the server keys_](#generate-the-server-keys).

4. Tell Wireguard that `wg0.conf` should define the configuration for the `wg0` interface:
    ```shell
    $ wg setconf wg0 /etc/wireguard/wg0.conf
    ```

5. Activate the `wg0` interface:
    ```shell
    $ ip link set up dev wg0
    ```
## Set up the client
Setting up a second peer 

I want to use a mobile phone as my client, which means I need to use the Wireguard mobile app to set up the tunnel. At a high level, I'm going to:
  1. On my server:
      * Generate private and public keys for my phone's Wireguard interface.
      * Create a configuration file called `mobile.conf`.
      * Use `qrencode` to generate a QR code from the config file.

  2. On my phone:
      * Download the Wireguard app.
      * Use the Wireguard app to scan the QR code and save the configuration.

### Generate the client keys
Generate another set of private and public keys. I prepended `mobile` to the names of these keys to indicate they're for my mobile phone.
```shell
$ wg genkey | tee mobileprivatekey | wg pubkey > mobilepublickey
```

### Set up the client interface
Create another config file. I called mine `mobile.conf` to indicate it's for my mobile phone.
```
[Interface]
Address = 192.168.2.2/28
PrivateKey = mobilepublickey

[Peer]
PublicKey = serverpublickey
Endpoint = <public domain/IP>:<port>
AllowedIPs = 192.168.2.1/32, 192.168.86.99/32
PersistentKeepalive = 15
```
Here's a breakdown of the configuration file:
* ``

### Add the client to the server configuration