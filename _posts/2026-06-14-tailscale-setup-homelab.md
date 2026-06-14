---
title: "Setting Up Tailscale for Your Homelab Infrastructure"
author:
  name: Łukasz Komosa
  link: https://github.com/komluk
date: 2026-06-14 13:00:00 +0000
categories: [Homelab, VPN, Networking]
tags: [tailscale, vpn, wireguard, homelab, self-hosted]
render_with_liquid: false
---

A while back I wrote about running [Zerotier on Linux](/posts/zerotier-on-linux/) to stitch machines together over the internet. `Tailscale` is a sibling tool that solves the same problem -- a flat, private network across hosts no matter where they live -- but it builds on `WireGuard` and wraps it in a polished identity and access-control model. In this post I'll walk through how I set Tailscale up across a homelab and the features that actually matter once you go beyond a single machine.

## What is Tailscale?

Tailscale is a zero-config mesh VPN built on top of `WireGuard`. You sign in on each device, and they all join a private network called a `tailnet`. Once a device is in the tailnet it can reach every other device by a stable `100.x.y.z` address (from the `100.64.0.0/10` CGNAT range) or by a friendly `MagicDNS` name -- regardless of NAT, firewalls, or which network the device is currently on. Connections are end-to-end encrypted and, where possible, direct peer-to-peer; only when a direct path can't be punched through does traffic fall back to a relay.

Compared to Zerotier, both are mesh VPNs with a coordination plane and per-device membership. The practical differences:

- **Tailscale** is `WireGuard`-based, ties identity to an SSO provider (Google, GitHub, etc.), and ships a clean `ACL` policy model and admin console.
- **Zerotier** runs its own protocol and leans more toward classic L2/L3 virtual-network semantics.

If you already like Zerotier, Tailscale will feel familiar -- just with WireGuard under the hood and a stronger focus on per-identity access rules.

## Installation and joining the tailnet

On any Linux host, the one-line installer does the work:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Then bring the node up and authenticate. This prints a URL you open in a browser to sign in with your SSO provider:

```bash
sudo tailscale up
```

Once authenticated, confirm the node is connected and see its peers:

```bash
tailscale status
```

To grab the node's tailnet IPv4 address:

```bash
tailscale ip -4
# 100.x.y.z
```

That `100.x.y.z` address is stable -- it stays with the node across reboots and network changes, which is exactly what you want for infrastructure.

## MagicDNS: names instead of numbers

Raw `100.x` addresses work, but nobody wants to memorize them. Enable `MagicDNS` in the admin console (DNS settings) and every node becomes reachable by name:

```bash
ssh homelab-server.tailnet-name.ts.net
```

The `tailnet-name.ts.net` suffix is assigned to your tailnet. After this, services and SSH configs can reference `homelab-server` instead of a numeric address that you'd otherwise have to keep straight by hand.

## Subnet routers: exposing a whole LAN

Not everything in a homelab can run the Tailscale client -- a NAS appliance, a printer, an IPMI/BMC interface, a switch. A `subnet router` lets one Tailscale node advertise an entire local subnet into the tailnet so those devices become reachable too.

On the node that sits on the homelab LAN:

```bash
sudo tailscale up --advertise-routes=10.0.0.0/24
```

Then approve the route in the admin console (advertised routes need explicit approval -- a nice safety gate). On the clients that should use it:

```bash
sudo tailscale up --accept-routes
```

Now any tailnet client can reach `10.0.0.50` (say, the NAS) through the subnet router, even though the NAS has no idea Tailscale exists.

**When to use a subnet router vs. installing the client:** if a device *can* run Tailscale, install the client directly -- you get end-to-end encryption all the way to that device and it shows up as a first-class node. Reserve subnet routers for things that can't run the client, or for bulk-exposing a lab segment you don't want to enroll device by device.

## Exit nodes

An `exit node` routes a client's *entire* internet traffic through one node. Advertise it from a host on the network you want to exit through:

```bash
sudo tailscale up --advertise-exit-node
```

Approve it in the admin console, then on a client select it (the GUI has a picker, or `tailscale up --exit-node=<node>`). This is handy when you're on untrusted Wi-Fi and want to egress through your home connection, or to reach services that only allow your home IP. Don't leave it on by default -- it sends all your traffic through that one hop.

## ACLs and tagged devices

By default everything in a tailnet can reach everything else. For a homelab that's usually too open. Tailscale's `ACL` policy file (written in `HuJSON` -- JSON with comments and trailing commas) is the source of truth for who can reach what. Lean on tags so policy follows roles, not individual machines.

A small, generic example:

```jsonc
{
  "tagOwners": {
    "tag:server": ["autogroup:admin"],
    "tag:laptop": ["autogroup:admin"],
  },
  "acls": [
    // Laptops can reach servers on SSH and HTTPS only.
    {
      "action": "accept",
      "src":    ["tag:laptop"],
      "dst":    ["tag:server:22,443"],
    },
    // Servers can talk to each other freely (e.g. monitoring scrape).
    {
      "action": "accept",
      "src":    ["tag:server"],
      "dst":    ["tag:server:*"],
    },
  ],
}
```

The goal is least privilege: grant the narrowest port and source set that still works, and widen only when something needs it.

Apply tags to nodes when they join:

```bash
sudo tailscale up --advertise-tags=tag:server
```

For headless or server installs, use an **auth key** generated in the admin console and bring the node up non-interactively. Pre-authorized, tagged nodes are ideal here -- a tagged node is owned by the tailnet rather than a person, so its key doesn't expire the way a user-authenticated node's does:

```bash
sudo tailscale up --authkey=tskey-auth-xxxxx --advertise-tags=tag:server
```

## Tailscale SSH (optional)

Tailscale can broker SSH itself, gating access through your ACLs instead of distributed key files:

```bash
sudo tailscale up --ssh
```

With a matching SSH rule in the policy file, you connect to `homelab-server` and Tailscale handles authentication -- no `authorized_keys` to sync, and access is revoked the instant the ACL changes. Useful for a fleet where key sprawl is the real maintenance burden.

## Hardening and operations

A few practices keep the tailnet trustworthy as it grows:

- **Key expiry.** User-authenticated nodes' keys expire on a schedule. For always-on servers, either disable key expiry per node in the admin console or run them as tagged nodes (which don't expire). Whatever you choose, plan rotation rather than getting surprised by a node dropping off.
- **Tailnet lock.** For higher assurance, `tailscale lock` (tailnet lock) requires that new nodes be cryptographically signed by trusted devices before they can join, so a compromised coordination plane can't silently add a node.
- **Least privilege in ACLs.** Default-deny and grant narrowly. Tags make this maintainable.
- **Admin console as source of truth.** Manage routes, exit nodes, key settings, and the policy file centrally -- don't let node-local flags drift from intent.
- **Only advertise what you need.** Don't leave a subnet router or exit node advertising routes you aren't actively using; every advertised route is attack surface and a routing surprise waiting to happen.

## Using it in practice

Once the tailnet is up, a lot of fragile homelab plumbing simply disappears. Instead of port-forwarding through your router and exposing services to the public internet, you reach them over the tailnet:

- Hit the Proxmox web UI at `https://homelab-server.tailnet-name.ts.net:8006` with no port forward.
- Reach monitoring dashboards, a self-hosted Git server, or the NAS by their MagicDNS names.
- Pair Tailscale with a reverse proxy so internal services get clean TLS names (`grafana.example.com`) that only resolve and route inside the tailnet.

The mental shift is that your homelab stops having a "public edge" you have to defend and instead becomes a private network you join -- access is an identity question, not a firewall-rule question.

## Takeaways

- Tailscale is a `WireGuard`-based mesh VPN; sign in on each device and they join a `tailnet` reachable by stable `100.x` IPs or MagicDNS names.
- Enable `MagicDNS` so you use names, not numbers.
- Use **subnet routers** for devices that can't run the client (NAS, printer, IPMI); install the client directly everywhere else.
- **Exit nodes** route all traffic through one node -- enable on demand, not by default.
- Model access with **ACLs and tags** for least privilege; use **auth keys / tagged nodes** for non-expiring headless servers.
- Harden with key-expiry discipline, optional tailnet lock, and keeping the admin console as the source of truth.
- In practice: reach Proxmox, monitoring, Git, and the NAS over the tailnet instead of port-forwarding, and pair with a reverse proxy for internal TLS names.
