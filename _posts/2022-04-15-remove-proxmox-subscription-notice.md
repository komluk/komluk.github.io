---
title: How to remove Proxmox Subscription Notice?
author:
  name: Łukasz Komosa
  link: https://github.com/komluk
date: 2022-04-15 12:00:00 +000
categories: [Proxmox]
tags: [proxmox, ssh]
render_with_liquid: false
---

`Proxmox Virtual Environment` is a popular open-source virtualization platform used by many organizations and individuals to run and manage virtual machines. While the `Proxmox` platform is free to use, it comes with a subscription notice that can be quite annoying. The subscription notice reminds users that they are using a free version of the platform and encourages them to purchase a subscription to access additional features and support. In this blog post, we'll show you how to remove the Proxmox subscription notice.

Before we get started, it's important to note that `Proxmox` relies on community support and contributions to maintain and improve the platform. If you find the platform useful, we encourage you to consider purchasing a subscription to support its continued development.

Now, let's get started with removing the `Proxmox` subscription notice.

## Step 1: SSH into your Proxmox server

The first step is to `SSH` into your `Proxmox` server. If you're using a Windows machine, you can use an `SSH` client like PuTTY. If you're using a Mac or Linux machine, you can use the built-in terminal.

## Step 2: Edit the subscription file

Once you're logged in, navigate to the `/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js` file using the following command:

```bash
nano /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
```

This will open the file in the nano text editor. Use the arrow keys to navigate to the following line:

```bash
if (data.status !== 'Active') {
```

## Step 3: Comment out the subscription notice

To remove the subscription notice, you need to comment out the line of code you located in `Step 2`. To do this, add two forward slashes at the beginning of the line, like this:

```bash
// if (data.status !== 'Active') {
```

## Step 4: Save the file

Once you've commented out the line, save the file by pressing `Ctrl + X`, then `Y`, then `Enter`.

## Step 5: Restart the Proxmox web service

Restart the `Proxmox` web service (also be sure to clear your browser cache, depending on the browser you may need to open a new tab or restart the browser)

```bash
systemctl restart pveproxy.service
```

## Step 6: Clear your browser cache

Finally, you need to clear your browser cache to see the changes. This can typically be done by pressing `Ctrl + Shift + R`.

And that's it! You should no longer see the `Proxmox` subscription notice. However, keep in mind that this is a temporary solution and may be overwritten during future `Proxmox` updates. To ensure a permanent solution, you may want to consider purchasing a `Proxmox` subscription.

In conclusion, removing the `Proxmox` subscription notice is a simple process that can be completed in just a few steps.However, we encourage users to consider purchasing a subscription to support the development of the `Proxmox` platform.
