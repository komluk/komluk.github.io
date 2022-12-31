---
title: What is ssh and how ssh-copy-id works?
author:
  name: Łukasz Komosa
  link: https://github.com/komluk
date: 2022-12-31 12:00:00 +000
categories: [ssh, Linux]
tags: [ssh, ssh-keygen, ssh-copy-id]
render_with_liquid: false
---

`SSH`, or Secure Shell, is a network protocol used to securely connect to a remote computer. It is commonly used for remote command-line access and remote command execution, but can also be used for secure network communication between two servers.

## How ssh works?
One important aspect of using `SSH` is the use of SSH keys, which are used to authenticate the client to the server. In order to use SSH keys for authentication, you first need to generate a public and private key pair on your local machine. The private key remains on your local machine and is used to authenticate your connection to the server. The public key, on the other hand, is copied to the server and is used to verify the authenticity of the private key.

## What is ssh-keygen?
`ssh-keygen` is a tool for creating new authentication key pairs for SSH. Such key pairs are used for automating logins, single sign-on, and for authenticating host

## Generate an SSH Key Pair on UNIX and UNIX-Like Systems
To generate an SSH key pair, you can use the `ssh-keygen` command. This command will prompt you for a location to save the key pair and a passphrase (optional). It is recommended to use a passphrase for added security. The `ssh-keygen` command will generate both a public and private key.

Run the ssh-keygen command. (You can use the -t option to specify the type of key to create.)

```bash
ssh-keygen -t rsa
```

You can use the -b option to specify the length (bit size) of the key, as shown in the following example:

```bash
ssh-keygen -t rsa -b 4096
```

## Copy ssh key on your server
Once you have generated your SSH key pair, you can copy the public key to the remote server by using the `ssh-copy-id` command. This command will copy your public key to the `~/.ssh/authorized_keys` file on the remote server, which is used to store the public keys of clients that are allowed to authenticate using SSH.

Here is an example of how to use `ssh-copy-id` to copy your public key to a remote server:
```bash
ssh-copy-id user@remote_server
```

Replace `user` with the username on the remote server and `remote_server` with the hostname or IP address of the remote server. You will be prompted for the password of the user on the remote server.

## Test your connection
Once you have copied your public key to the remote server, you should be able to use your private key to authenticate to the server using SSH. To do so, you can use the `ssh` command with the `-i` option to specify the path to your private key:

```bash
ssh -i /path/to/private_key user@remote_server
```

Replace `/path/to/private_key` with the path to your private key file and `user` and `remote_server` with the appropriate values for your setup.

In summary, SSH keys provide a secure and convenient way to authenticate to a remote server using SSH. To use SSH keys, you need to generate a public and private key pair on your local machine and copy the public key to the remote server using the `ssh-copy-id` command. You can then use your private key to authenticate to the remote server using the `ssh` command with the `-i` option.