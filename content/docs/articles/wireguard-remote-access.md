---
title: Access a remote server from anywhere via Wireguard
weight: 1
---
# Access a remote server from anywhere via Wireguard

## Introduction
I self-host a few apps on my home network, using a small single-board computer as my server.

Until recently, I was only able to access my self-hosted apps at home on my home network. That limitation caused me a few problems.

For example, one of the apps I self-host is [Actual](https://actualbudget.org/), an open-source budgeting app that totally changed the way I manage my personal finances. I love Actual, and I love the idea of keeping my financial data on my home network away from prying eyes. However, my inability to access Actual remotely meant I couldn't update my budget or add transactions without being at home. Annoying!

My solution? Use [Wireguard](https://www.wireguard.com/) to create a secure VPN tunnel to my home server, allowing me to access my self-hosted apps from anywhere.

## Use case
- I only care about accessing resources on a single server. I don't need remote access to anything else on my home network.

- I plan to use my mobile phone as the "client" to access my server.

- I don't need to send all my internet traffic through the VPN tunnel, just the traffic to and from my server.

- My server is running [Ubuntu 20.04.6 LTS](https://releases.ubuntu.com/focal/) on the ARMv7 architecture.

{{<hint warning>}}
**A note on terminology**

In this article, I use the terms _server_ and _client_ to describe the two endpoints of a Wireguard tunnel. I also use the Wireguard term _peer_ to refer to these endpoints in the abstract.
{{</hint>}}

## Set up the server

### Install Wireguard on the server
Use the [installation method](https://www.wireguard.com/install/) of your choice to install Wireguard. I'm on Ubuntu, so I use the `apt` package manager.
```shell
$ sudo apt install wireguard
```
Test that Wireguard installed successfully by running the [`wg`](https://man7.org/linux/man-pages/man8/wg.8.html) command. No output means the installation was successful.

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

### Configure the server interface
With the private and public keys ready, it's time to set up the Wireguard interface on the server.

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

I want to use a mobile phone as my client, which means I need to use the Wireguard mobile app to set up the tunnel. Here's the high level process:
  1. On the server:
      * Generate private and public keys for the phone's Wireguard interface.
      * Create a configuration file called `mobile.conf`.
      * Use `qrencode` to generate a QR code from the config file.

  2. On the phone:
      * Download the Wireguard app.
      * Use the Wireguard app to scan the QR code and save the configuration.

### Generate the client keys
Generate another set of private and public keys. I prepended `mobile` to the names of these keys to indicate they're for my mobile phone.
```shell
$ wg genkey | tee mobileprivatekey | wg pubkey > mobilepublickey
```

### Configure the client interface
Create another config file in `/etc/wireguard` to set up the client interface. I called mine `mobile.conf` to indicate it's for my mobile phone. The config file should also include information about the client's _peer_ — the server, in this case.
```
[Interface]
Address = 192.168.2.2/28
PrivateKey = mobileprivatekey

[Peer]
PublicKey = serverpublickey
Endpoint = mydomain.duckdns.org:51820
AllowedIPs = 192.168.2.1/32, 192.168.86.99/32
```
The `[Peer]` section tells the client interface how to connect to the server interface. Here's a breakdown:
* `PublicKey`: The server's public key
* `Endpoint`: A publically accessible domain name or IP address<sup>1</sup> for the server, plus the `ListenPort` specified in the server interface configuration<sup>2</sup>. 
* `AllowedIPs`: The IP addresses that the client should be able to access, expressed in CIDR notation. In my case, the client only needs to connect to the server, so I specified the IP address of the server's Wireguard interface and the IP address of the server on my home network.

  <sup>1</sup> The IP address assigned by my ISP is dynamic, so I use a [DuckDNS](https://www.duckdns.org/) domain name that always points to my public IP, even when it changes.

  <sup>2</sup> You'll need to set up port forwarding on the server's network gateway to forward traffic on this port to the `ListenPort` defined in the server's interface configuration.

### Add the client to the server configuration
Now that you've set up the client configuration, go back to the server configuration in `wg0.conf` and add information about the client to the bottom of the file.
```
[Peer]
PublicKey = mobilepublickey
AllowedIPs = 192.168.2.2/32
```
* `PublicKey`: The client's public key
* `AllowedIPs`: The IP addresses that the server interface should be able to send packets to, expressed in CIDR notation. In my case, there's only one: the IP address of the client interface defined in `mobile.conf`.


### Generate a QR code
1. Download `qrencode`:
    ```shell
    $ sudo apt install qrencode
    ```
2. Use `qrencode` to generate a QR code from `mobile.conf`. My server is headless — it has no GUI — so I used the `ansiutf8` option to specify that the QR code should be in a plaintext format.
    ```shell
    $ qrencode -t ansiutf8 < /etc/wireguard/mobile.conf
    ```

## Start the server interface
1. On the server, use the following command to start the `wg0` interface:
    ```shell
    $ wg-quick up wg0
    ```
2. Start the Wireguard service:
    ```shell
    $ sudo systemctl start wg-quick@wg0.service
    ```

## Set up the mobile interface
1. Download the Wireguard mobile app.

2. In the Wireguard mobile app, add a new tunnel and select **Create a new QR code**.

2. On the server, open the QR code. If you have a plaintext QR code, you can use `cat` or a text editor like `nano` to display it.
    ```shell
    $ cat /etc/wireguard/mobile.conf
    ```
3. Scan the QR code and save the settings.

## Connect to the server from the client
At this point, you should have a working tunnel configured between your server and client. Use the Wireguard mobile app to start the client interface and enjoy accessing your server from anywhere!