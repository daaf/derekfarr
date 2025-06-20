---
title: Access a remote server from a mobile phone via WireGuard
weight: 3
---
# Access a remote server from a mobile phone via WireGuard

## Introduction
I self-host a few apps on my home network and needed a way to access those apps remotely. My solution? Use [WireGuard](https://www.wireguard.com/) to create a secure <abbr title="Virtual Private Network">VPN</abbr> tunnel to my home server, allowing me to access my self-hosted apps from anywhere.

This article describes how to create a bare-bones WireGuard setup that satisfies my basic requirements:
- I need access to resources on a single server. I don't need to access anything else on my home network or on the internet through the tunnel.
- I must be able to use my phone as the "client" to access the server.

My server is running [Ubuntu 20.04.6 LTS](https://releases.ubuntu.com/focal/) on the ARMv7 architecture.

{{<hint warning>}}
**A note on terminology**

In this article, I use the terms _server_ and _client_ to describe the two endpoints of a WireGuard tunnel. I also use the WireGuard term _peer_ to refer to these endpoints in the abstract.
{{</hint>}}

## Instructions

### Install WireGuard
Install WireGuard on the server using the [installation method](https://www.wireguard.com/install/) of your choice. I'm on Ubuntu, so I use the `apt` package manager:
```shell
$ sudo apt install wireguard
```
Test that WireGuard installed successfully by running the [`wg`](https://man7.org/linux/man-pages/man8/wg.8.html) command. No output means the installation was successful.

### Generate keys
WireGuard relies on encrypted keys for authentication between peers. Each peer needs to have a private key and an associated public key that's derived from the private key.

1. Create a private and public key for the server with the following command:
    ```shell
    $ wg genkey | tee privatekey | wg pubkey > publickey
    ```
    * `wg genkey`: Generates a private key
    * `tee privatekey`: Saves the private key in a file called `privatekey` and writes the private key to stdout.
    * `wg pubkey > publickey`: Generates a public key from the private key and saves the public key in a file called `publickey`

2. Generate another set of private and public keys, this time for the client. Give the key files a unique name. For example, prepend `mobile` to indicate the keys are for a mobile phone.
    ```shell
    $ wg genkey | tee mobileprivatekey | wg pubkey > mobilepublickey
    ```
With the private and public keys ready, you're ready to configure the WireGuard tunnel. To do so, you'll create two configuration files, one for the server and one for the client.

### Configure the server

1. On the server, create a config file for an interface called `wg0` at `/etc/wireguard/wg0.conf`. Add the following configuration:
    ```
    [Interface]
    Address = 192.168.2.1/32
    ListenPort = 51820
    PrivateKey = serverprivatekey
    ```
    * `Address`: An IP address and subnet for the server's interface on WireGuard's virtual network. It shouldn't overlap with any subnet in use on the server's <abbr title="Local Area Network">LAN</abbr>. In this basic use case with only two peers, the interface's subnet consists of a single IP address.
    * `ListenPort`: The port the interface is listening on.
    * `PrivateKey`: The **server** private key you generated in [_Generate the server and client keys_](#generate-the-server-and-client-keys).

2. Restrict the permissions of `wg0.conf`:
    ```shell
    $ chmod 600 /etc/wireguard/wg0.conf
    ```

### Configure the client
1. Create another config file at `/etc/wireguard/mobile.conf` for the client configuration.
    ```
    [Interface]
    Address = 192.168.2.2/32
    PrivateKey = mobileprivatekey

    [Peer]
    PublicKey = serverpublickey
    Endpoint = mydomain.duckdns.org:51820
    AllowedIPs = 192.168.2.1/32, 192.168.86.99/32
    ```
    The `[Interface]` section defines the client interface.
    * `Address`: An IP address and subnet for the client's interface on WireGuard's virtual network. Like in the server interface configuration, the subnet here is just a single IP address.
    * `PrivateKey`: The **client** private key you generated in [_Generate server and client keys_](#generate-server-and-client-keys).

    The `[Peer]` section tells the client interface how to connect to the server interface.
    * `PublicKey`: The server's public key.
    * `Endpoint`: A publicly accessible domain name or IP address<sup>1</sup> for the server, plus the `ListenPort` specified in the server interface configuration<sup>2</sup>. 
    * `AllowedIPs`: IP addresses that the client should accept traffic from over its WireGuard interface. To connect to resources on my home server, I entered two IP addresses:
        * The IP address of the server's WireGuard interface defined in `wg0.conf`
        * The IP address of the server on my LAN at home

    <sup>1</sup> The IP address assigned by my <abbr title="Internet Service Provider">ISP</abbr> is dynamic, so I use a [DuckDNS](https://www.duckdns.org/) domain name that always points to my public IP, even when it changes.

    <sup>2</sup> You'll need to set up port forwarding on the server's network gateway to forward traffic on this port to the `ListenPort` defined in the server's interface configuration.

2. Restrict the permissions of `mobile.conf`:
    ```shell
    $ chmod 600 /etc/wireguard/mobile.conf
    ```

2. Go back to the server configuration in `wg0.conf` and create a `Peer` section with information about the client.
    ```
    [Peer]
    PublicKey = mobilepublickey
    AllowedIPs = 192.168.2.2/32
    ```
    * `PublicKey`: The client's public key
    * `AllowedIPs`: IP addresses that the server interface should be able to send packets to. In this case, there's only one: the IP address of the client's WireGuard interface defined in `mobile.conf`.

At this point it might be helpful to review the contents of the two configuration files side-by-side:
{{<columns>}}
```
/etc/wireguard/wg0.conf

[Interface]
Address = 192.168.2.1/32
ListenPort = 51820
PrivateKey = serverprivatekey

[Peer]
PublicKey = mobilepublickey
AllowedIPs = 192.168.2.2/32
```
<--->
```
/etc/wireguard/mobile.conf

[Interface]
Address = 192.168.2.2/32
PrivateKey = mobileprivatekey

[Peer]
PublicKey = serverpublickey
Endpoint = mydomain.duckdns.org:51820
AllowedIPs = 192.168.2.1/32, 192.168.86.99/32
```
{{</columns>}}

In the server configuration, the peer's `AllowedIPs` value is the same as the client interface's `Address`. Likewise, in the client configuration, one of the peer's `AllowedIPs` is  the server interface's `Address`.

### Set up the client
At this point, you've created the server and client configuration and have the server interface running. Now you need to get the client configuration onto your phone.

1. On the server, download `qrencode`:
    ```shell
    $ sudo apt install qrencode
    ```
2. Use `qrencode` to generate a QR code from `mobile.conf`. My server has no desktop environment, so I used the `ansiutf8` option to specify that the QR code should be in a plain text format.
    ```shell
    $ qrencode -t ansiutf8 < /etc/wireguard/mobile.conf
    ```
3. Open the QR code. If you have a plain text QR code, you can use `cat` or a text editor like `nano` to display it.
    ```shell
    $ cat /etc/wireguard/mobile.conf
    ```

4. On your phone, download the WireGuard mobile app.

5. In the WireGuard mobile app, add a new tunnel and select **Create from QR code**.

6. Use your phone to scan the QR code and save the settings.

### Start the interfaces and connect
1. On the server, add the WireGuard service to `systemd` so the `wg0` interface starts automatically when the server boots up:
    ```shell
    $ sudo systemctl enable wg-quick@wg0.service
    $ sudo systemctl daemon-reload
    ```
2. Start the WireGuard service:
    ```shell
    $ sudo systemctl start wg-quick@wg0.service
    ```
3. On your phone, use the WireGuard mobile app to activate the client interface.

You should now have an active, encrypted WireGuard tunnel between your server and phone. Enjoy accessing your server from anywhere!