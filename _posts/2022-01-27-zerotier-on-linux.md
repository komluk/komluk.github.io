---
title: Installing and Using Zerotier VPN on a Linux Machine
author:
  name: Łukasz Komosa
  link: https://github.com/komluk
date: 2022-01-27 23:00:00 +000
categories: [VPN, Zerotier, Linux]
tags: [VPN]
render_with_liquid: false
---

Zerotier is a popular virtual private network `VPN` service that allows users to connect their devices together securely over the internet. It is easy to set up and use, and offers a range of features such as network bridging, remote access, and support for multiple protocols. In this tutorial, we will walk through the process of installing and using `Zerotier` on a Linux machine.

## Prerequisites
Before you can install Zerotier, you will need to ensure that your Linux machine meets the following requirements:

- A 64-bit processor
- At least 4GB of RAM
- An internet connection

## Installing Zerotier
To install Zerotier on your Linux machine, follow these steps:

If you`re willing to rely on SSL to authenticate the site, a one line install can be done with:

```bash
curl -s https://install.zerotier.com | sudo bash
```

If you have GPG installed, a more secure option is available:

```bash
curl -s 'https://raw.githubusercontent.com/zerotier/ZeroTierOne/master/doc/contact%40zerotier.com.gpg' | gpg --import && \
if z=$(curl -s 'https://install.zerotier.com/' | gpg); then echo "$z" | sudo bash; fi
```

Once the installation is complete, start the Zerotier service using the following command:

```bash
sudo systemctl start zerotier-one
```

Enable the Zerotier service to start automatically at boot time using the following command:

```bash
sudo systemctl enable zerotier-one
```

## Joining a Zerotier Network
To join a Zerotier network, you will need to install Zerotier on the device that you want to connect to the network. Follow the instructions in the "Installing Zerotier" section above to install Zerotier on your device.


Once Zerotier is installed, you can join a network by following these steps:

Get your ZeroTier address and check the service status

```bash
zerotier-cli status
```

Join, leave, and list networks. Remember, ZeroTier networks are 16-digit IDs that look like 8056c2e21c000001

```bash
zerotier-cli join ################
```

```bash
zerotier-cli leave ################
```

```bash
zerotier-cli listnetworks
```