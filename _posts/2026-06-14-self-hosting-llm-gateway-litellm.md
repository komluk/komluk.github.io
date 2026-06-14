---
title: "Self-Hosting an LLM Gateway With LiteLLM"
author:
  name: Łukasz Komosa
  link: https://github.com/komluk
date: 2026-06-14 12:00:00 +0000
categories: [Homelab, AI, Self-Hosted]
tags: [litellm, llm, homelab, self-hosted, proxy]
render_with_liquid: false
---

If you run more than one tool that talks to a language model -- a CLI here, a script there, a couple of agents -- you quickly end up with API keys scattered across config files, hardcoded model names, and no way to see what is actually costing you money. A self-hosted `LLM gateway` fixes that by putting a single, OpenAI-compatible endpoint in front of every provider and model you use. In this post I'll walk through why a gateway is worth running in a homelab and how to stand one up with `LiteLLM`.

## Why a gateway?

Instead of every app holding its own provider key and model name, they all point at one URL. That gives you a few concrete wins:

- **One endpoint, many models.** Apps hit a single OpenAI-compatible `/v1/chat/completions`, and the gateway routes to whichever provider backs that model.
- **Centralized keys.** Provider secrets live in one place, not copy-pasted into a dozen `.env` files.
- **Swap models without touching apps.** Point a friendly name like `fast-model` at a different backend and every client follows -- no redeploys.
- **Budgets and rate limits.** Cap spend and throughput per app, so a runaway script can't drain your wallet.
- **Logging and a single auth point.** One place to see requests, and one master key to rotate.

This is especially handy when several homelab tools (CLIs, cron scripts, local agents) all need model access and you want to govern them as a group rather than one at a time.

## Architecture

The setup is deliberately boring, which is what you want for infrastructure:

- `LiteLLM` runs as a proxy at the core, listening on a local port (default `4000`).
- A reverse proxy (`Caddy` or `Nginx`) sits in front of it to terminate `TLS`.
- Everything lives in a container or `LXC`, so it's easy to snapshot and rebuild.
- The proxy exposes an OpenAI-compatible API, so existing SDKs and tools "just work" -- you only change the base URL and the key.

```
client ──HTTPS──> Caddy (TLS) ──HTTP──> LiteLLM :4000 ──> provider APIs
```

## A minimal config

`LiteLLM` is driven by a `config.yaml`. At its simplest you map a friendly model name to a provider model and set a master key. Keep real secrets out of the file by reading them from the environment.

```yaml
model_list:
  - model_name: fast-model
    litellm_params:
      model: openai/gpt-4o-mini
      api_key: <YOUR_PROVIDER_KEY>

  - model_name: smart-model
    litellm_params:
      model: anthropic/claude-3-5-sonnet
      api_key: <YOUR_PROVIDER_KEY>

litellm_settings:
  drop_params: true
  request_timeout: 600

general_settings:
  master_key: <YOUR_MASTER_KEY>
```

Clients only ever see `fast-model` and `smart-model`. The provider behind each name is your business, and you can change it without telling a single app.

## Running it

The quickest way to run the proxy is the official container image. Mount your config and pass the port through:

```bash
docker run -d \
  --name litellm \
  -p 4000:4000 \
  -v $(pwd)/config.yaml:/app/config.yaml \
  ghcr.io/berriai/litellm:main-latest \
  --config /app/config.yaml
```

If you prefer `docker-compose`, the equivalent is short:

```yaml
services:
  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    command: ["--config", "/app/config.yaml"]
    ports:
      - "4000:4000"
    volumes:
      - ./config.yaml:/app/config.yaml
    restart: unless-stopped
```

Once it's up, test it like any OpenAI endpoint. Use your friendly model name and the master key as the bearer token:

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer <YOUR_MASTER_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "fast-model",
    "messages": [{"role": "user", "content": "Say hello from my homelab."}]
  }'
```

Any tool that speaks the OpenAI API will work the same way: set the base URL to your gateway and the API key to a gateway key.

## Adding TLS

You don't want to send keys and prompts over plain HTTP, even on a LAN. `Caddy` makes TLS almost free thanks to automatic certificates. A minimal `Caddyfile` is two lines:

```
llm.example.com {
    reverse_proxy localhost:4000
}
```

Point `llm.example.com` at the gateway, and clients now connect over HTTPS while `LiteLLM` keeps listening on plain HTTP behind it.

## Operational touches

Once the basics work, a few features make the gateway genuinely useful day to day:

- **Virtual keys per app.** Instead of handing out the master key, mint a separate key for each tool with its own budget and rate limit. If one leaks, you revoke just that one.

```bash
curl http://localhost:4000/key/generate \
  -H "Authorization: Bearer <YOUR_MASTER_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"models": ["fast-model"], "max_budget": 5, "duration": "30d"}'
```

- **Request logging.** Enable logging or a database backend so you can see who called what, how many tokens it cost, and where the spend is going.
- **Fallbacks and load balancing.** List the same `model_name` multiple times with different backends, and `LiteLLM` will load-balance across them and fail over if one provider is down.

```yaml
litellm_settings:
  fallbacks:
    - smart-model: ["fast-model"]
```

## Honest caveats

Self-hosting means you own the boring parts too:

- **You own uptime and security.** If the gateway is down, every tool that depends on it is down. Snapshot the container and keep the config in version control.
- **Protect the master key.** It can mint keys and read everything. Keep it in a secrets store or environment variable, never in a committed file, and rotate it periodically.
- **Rate-limit aggressively.** A misbehaving agent in a loop can burn through a provider budget fast. Per-key limits are your safety net.
- **Keep it off the public internet unless you really need it.** The simplest secure setup is to reach the gateway over a private overlay network rather than exposing it to the world. A VPN like `Tailscale` or `Zerotier` works well here -- I covered setting up Zerotier on Linux in an earlier post, and the same network is a natural home for a private gateway.

That's the whole idea: one governed front door for every model in your homelab. Start with a single model and the master key, confirm a `curl` works, then layer on virtual keys, logging, and TLS as your needs grow.
