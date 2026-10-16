# Fuiz Self-Hosted

Self-hosting [Fuiz](https://fuiz.org) using Docker.

This repository contains configurations and instructions for deploying a full Fuiz instance, including the web frontend, game server, and image storage server.

> [!NOTE]
> This guide is inspired by [Stoat's self-hosted setup](https://github.com/stoatchat/self-hosted).

## Table of Contents

- [Deployment](#deployment)
  - [Configure your domain](#configure-your-domain)
  - [Install required dependencies](#install-required-dependencies)
- [Configuration](#configuration)
  - [AI (optional)](#ai-optional)
  - [Library publishing and Google Drive backups (optional)](#library-publishing-and-google-drive-backups-optional)
- [Updating](#updating)
- [Additional Notes](#additional-notes)
  - [Architecture](#architecture)
  - [Local image builds](#local-image-builds)
  - [Data and backups](#data-and-backups)

## Deployment

To get started, find a suitable server to deploy onto. Fuiz is lightweight and should run comfortably on most hardware.

### Configure your domain

Your domain (or a subdomain) should point to the server's IP address using A and AAAA records.

If you just want to try it locally, you can use `localhost` instead.

### Install required dependencies

You'll need Git and Docker (with the Compose plugin) installed:

```bash
# Install Git and Docker (Ubuntu example)
apt-get update
apt-get install ca-certificates curl git
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Configuration

Clone this repository and enter the directory:

```bash
git clone https://gitlab.com/fuiz/self-hosted fuiz
cd fuiz
```

Generate a configuration file by running:

```bash
chmod +x ./generate_config.sh

# For a public domain (Caddy handles HTTPS automatically)
./generate_config.sh fuiz.example.com

# Or for local development
./generate_config.sh localhost
```

This creates a `.env` file with all the necessary settings. You can also copy `.env.example` and edit it manually.

Start Fuiz in the foreground first to check for errors:

```bash
docker compose up
```

If it runs without any critical errors, you can stop it with <kbd>Ctrl</kbd> + <kbd>C</kbd> and run it detached (in the background) by appending `-d`:

```bash
docker compose up -d
```

Your Fuiz instance should now be available at `https://your.domain` (or `https://localhost` for local).

On `localhost`, Caddy signs the certificate with its own local authority, so browsers show a warning the first time. Accept it, or trust Caddy's root certificate on your machine:

```bash
docker compose cp caddy:/data/caddy/pki/authorities/local/root.crt caddy-root.crt
```

Then import `caddy-root.crt` into your system or browser certificate store.

### AI (optional)

Fuiz can use an AI model to automatically derive tags for library quizzes. This is optional and disabled by default.

To enable it, uncomment and configure the AI variables in your `.env` file:

```bash
OPENAI_BASE_URL=http://localhost:11434/v1   # Ollama example
OPENAI_API_KEY=                              # Not needed for Ollama
AI_MODEL=gpt-4o-mini
```

Any OpenAI-compatible API works: OpenAI, Ollama, OpenRouter, LiteLLM, etc.

### Library publishing and Google Drive backups (optional)

Both features are off unless configured, and both take their settings from the same `.env` file:

- Publishing quizzes to a public library needs a GitLab application and a library repository. See [GITLAB_PUBLISHING_SETUP.md](https://gitlab.com/fuiz/website/-/blob/main/GITLAB_PUBLISHING_SETUP.md).
- Google Drive backups need a Google application. See the [website README](https://gitlab.com/fuiz/website#developing) for the variables.

Restart with `docker compose up -d` after editing `.env`.

## Updating

Pull the latest version of this repository:

```bash
git pull
```

Your `.env` is not touched by `git pull`. If an update adds new settings, compare your `.env` with `.env.example`, or regenerate it. The configuration script refuses to replace an existing `.env` unless you pass `--overwrite`, and then keeps the previous file as `.env.old`:

```bash
./generate_config.sh --overwrite your.domain
```

Copy any settings you added by hand, such as AI or publishing settings, from `.env.old` into the new `.env`.

Then pull all the latest images and restart:

```bash
docker compose pull
docker compose up -d
```

## Additional Notes

### Architecture

The stack consists of four services:

| Service         | Description                                                       |
| --------------- | ----------------------------------------------------------------- |
| **caddy**       | Reverse proxy with automatic HTTPS via Let's Encrypt              |
| **web**         | SvelteKit frontend (uses SQLite and local filesystem for storage) |
| **game-server** | Game backend (Rust/actix-web)                                     |
| **corkboard**   | Image storage server (filesystem-backed with in-memory cache)     |

Caddy routes requests as follows:

- `/api/*` to game-server
- `/corkboard/*` to corkboard
- Everything else to web

### Local image builds

For development or testing with locally built images, edit `compose.override.yml`:

```yaml
services:
  web:
    image: fuiz-web:test
  game-server:
    image: fuiz-server:test
  corkboard:
    image: fuiz-corkboard:test
```

Docker Compose automatically picks up `compose.override.yml` alongside `compose.yml`.

### Data and backups

Persistent data lives in Docker volumes:

| Volume           | Contents                                      |
| ---------------- | --------------------------------------------- |
| `web-data`       | SQLite database, KV store, and uploaded media |
| `corkboard-data` | Images stored by the corkboard service        |
| `caddy-data`     | TLS certificates                              |
| `caddy-config`   | Caddy configuration state                     |

To back up your data, use `docker compose cp` or back up the volume directories directly.
