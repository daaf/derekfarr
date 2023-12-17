---
title: Access a remote server from a mobile phone via Wireguard
weight: 1
---
# Access a remote server from a mobile phone via Wireguard

## Introduction
I self-host a few apps on my home network, using a small single-board computer as my server.

Until recently, I was only able to access my self-hosted apps at home on my home network. That limitation caused me a few problems.

For example, one of the apps I self-host is [Actual](https://actualbudget.org/), an open-source budgeting app that totally changed the way I manage my personal finances. I love Actual, and I love the idea of keeping my financial data on my home network away from prying eyes. However, my inability to access Actual remotely meant I couldn't update my budget or add transactions without being at home. Annoying!

My solution? Use [Wireguard](https://www.wireguard.com/) to create a secure <abbr title="Virtual Private Network">VPN</abbr> tunnel to my home server, allowing me to access my self-hosted apps from anywhere.

## Use case
- I only care about accessing resources on a single server. I don't need remote access to anything else on my home network.

- I plan to use my phone as the "client" to access my server.

- I don't need to send all my internet traffic through the VPN tunnel, just the traffic to and from my server.

- My server is running [Ubuntu 20.04.6 LTS](https://releases.ubuntu.com/focal/) on the ARMv7 architecture.

{{<hint warning>}}
**A note on terminology**

In this article, I use the terms _server_ and _client_ to describe the two endpoints of a Wireguard tunnel. I also use the Wireguard term _peer_ to refer to these endpoints in the abstract.
{{</hint>}}

## Instructions

### Install Wireguard
Install Wireguard on the server
Use the [installation method](https://www.wireguard.com/install/) of your choice to install Wireguard. I'm on Ubuntu, so I use the `apt` package manager.
```shell
$ sudo apt install wireguard
```
Test that Wireguard installed successfully by running the [`wg`](https://man7.org/linux/man-pages/man8/wg.8.html) command. No output means the installation was successful.

### Generate server and client keys
Wireguard relies on encrypted keys for authentication between peers. Each peer needs to have a private key and an associated public key that's derived from the private key.

Conveniently, Wireguard comes with a few commands to create and store private and public keys. You can even pipe those commands together!

1. Create a private and public key for the server with the following command:
    ```shell
    $ wg genkey | tee privatekey | wg pubkey > publickey
    ```
    * `wg genkey`: Generates a private key
    * `tee privatekey`: Saves the private key in a file called `privatekey` and writes the private key to stdout.
    * `wg pubkey > publickey`: Generates a public key from the private key and saves the public key in a file called `publickey`

2. Generate another set of private and public keys, this time for the client. Give the key files a unique name. I prepended `mobile` to the key names to indicate they're for my phone.
    ```shell
    $ wg genkey | tee mobileprivatekey | wg pubkey > mobilepublickey
    ```
With the private and public keys ready, you're ready to configure the Wireguard tunnel. To do so, you'll create two configuration files, one for the server and one for the client.

### Configure the server

1. On the server, create a config file for the `wg0` interface at `/etc/wireguard/wg0.conf`. Add the following configuration:
    ```
    [Interface]
    Address = 192.168.2.1/28
    SaveConfig = true
    ListenPort = 51820
    PrivateKey = serverprivatekey
    ```
    * `Address`: An IP address to assign to the interface. It should be in the range defined in [Create the server interface](create-the-server-interface)
    * `SaveConfig`: Whether the configuration should be saved after the interface is shut down
    * `ListenPort`: The port the interface is listening on
    * `PrivateKey`: The private key you generated in [_Generate the server keys_](#generate-the-server-keys)

2. Tell Wireguard that `wg0.conf` should define the configuration for the `wg0` interface:
    ```shell
    $ wg setconf wg0 /etc/wireguard/wg0.conf
    ```

### Configure the client
1. Create another config file at `/etc/wireguard/mobile.conf` for the client configuration.
    ```
    [Interface]
    Address = 192.168.2.2/28
    PrivateKey = mobileprivatekey

    [Peer]
    PublicKey = serverpublickey
    Endpoint = mydomain.duckdns.org:51820
    AllowedIPs = 192.168.2.1/32, 192.168.86.99/32
    ```
    The config file defines the client interface and includes information about the client's peer: the server.

    The `[Interface]` section defines the client interface.
    * `Address`: An IP address to assign to the interface. It should be in the range defined in [_Create the server interface_](create-the-server-interface)
    * `PrivateKey`: The **client** private key you generated in [_Generate server and client keys_](#generate-server-and-client-keys)

    The `[Peer]` section tells the client interface how to connect to the server interface.
    * `PublicKey`: The server's public key
    * `Endpoint`: A publicly accessible domain name or IP address<sup>1</sup> for the server, plus the `ListenPort` specified in the server interface configuration<sup>2</sup>. 
    * `AllowedIPs`: The IP addresses that the client should be able to access, expressed in <abbr title="Classless Inter-Domain Routing">CIDR</abbr> notation. In my case, the client only needs to connect to the server, so I specified the IP address of the server's Wireguard interface and the IP address of the server on my home network.

    <sup>1</sup> The IP address assigned by my <abbr title="Internet Service Provider">ISP</abbr> is dynamic, so I use a [DuckDNS](https://www.duckdns.org/) domain name that always points to my public IP, even when it changes.

    <sup>2</sup> You'll need to set up port forwarding on the server's network gateway to forward traffic on this port to the `ListenPort` defined in the server's interface configuration.

2. Go back to the server configuration in `wg0.conf` and create a `Peer` section with information about the client.
    ```
    [Peer]
    PublicKey = mobilepublickey
    AllowedIPs = 192.168.2.2/32
    ```
    * `PublicKey`: The client's public key
    * `AllowedIPs`: The IP addresses that the server interface should be able to send packets to, expressed in CIDR notation. In this case, there's only one: the IP address of the client interface defined in `mobile.conf`.

### Start the server interface
1. Use the following command to start the `wg0` interface:
    ```shell
    $ wg-quick up wg0
    ```
2. Start the Wireguard service:
    ```shell
    $ sudo systemctl start wg-quick@wg0.service
    ```

### Generate a QR code from the client config
1. Download `qrencode`. On Ubuntu:
    ```shell
    $ sudo apt install qrencode
    ```
2. Use `qrencode` to generate a QR code from `mobile.conf`. My server is headless—it has no GUI—so I used the `ansiutf8` option to specify that the QR code should be in a plain text format.
    ```shell
    $ qrencode -t ansiutf8 < /etc/wireguard/mobile.conf
    ```

### Set up the client
At this point, you've created the server and client configuration and have the server interface running. Now you need to get the client configuration onto your phone.

1. On your phone, download the Wireguard mobile app.

2. In the Wireguard mobile app, add a new tunnel and select **Create a new QR code**.

2. On the server, open the QR code. If you have a plain text QR code, you can use `cat` or a text editor like `nano` to display it.
    ```shell
    $ cat /etc/wireguard/mobile.conf
    ```
3. Use your phone to scan the QR code and save the settings.

### Connect to the server from the client
Use the Wireguard mobile app to start the client interface. You should now have an active, encrypted Wireguard tunnel between your server and phone. Enjoy accessing your server from anywhere!