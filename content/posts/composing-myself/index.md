---
title: "Composing Myself"
summary: "From a simple ad-blocker to a fully automated home lab, here's a technical deep-dive into my Raspberry Pi's Docker Compose setup."
date: "2025-06-06"
series: ["RaspberryPi"]
series_order: 2
---

## The Goal

In my last post, ["Working 22/7,"]({{< ref "posts/working-22-7" >}}) I introduced my [Raspberry Pi](https://www.raspberrypi.com/) home lab. The goal was to build a platform on a simple device that was powerful yet resilient, and easy to tinker with. The key constraints were cost and physical size, but the most important requirement was stability. I needed peace of mind, with no risk of a failed experiment breaking the internet and facing the wrath of a Wi-Fi-less family.

## The Setup

The key to this entire setup is defining the whole stack as code, orchestrated by [Docker](https://www.docker.com/). The secret lies in a single file: `docker-compose.yml`. Think of it as the blueprint for the entire project. It tells Docker exactly which services to run, how to configure their network, and where to store their data, ensuring the setup is repeatable and easy to manage.

{{< codeurl url="https://raw.githubusercontent.com/geeksbsmrt/RaspberryPi/refs/heads/main/docker/docker-compose.yml" lang="yml"  >}}

> [!TIP] A Note on Networking
> I use a [macvlan](https://docs.docker.com/network/macvlan/) network for most of the services. This gives each container its own unique IP address on my home network, just like a physical device. This avoids port conflicts between services and makes network configuration much cleaner.

### The Core Services

These are the foundational services that provide core functionality for my network and the services I host.

#### Pi-hole

[Pi-hole](https://pi-hole.net/) is the cornerstone of my network management. Its first job is as a network-wide DNS sinkhole, effectively blocking ads and trackers for every device. Its second crucial role is as my network's DHCP server, which gives me granular control over IP address assignment.

#### Caddy

[Caddy](https://caddyserver.com/) is my reverse proxy of choice. It's lightweight, incredibly easy to configure, and its killer feature is fully automated HTTPS. It handles all my TLS certificate acquisition and renewal from [Let's Encrypt](https://letsencrypt.org/) without any manual intervention. I use the [Cloudflare](https://www.cloudflare.com/) plugin to solve DNS challenges, which lets me secure internal services that aren't directly exposed to the internet.

#### Umami

Since I'm running a blog, I wanted privacy-focused analytics. [Umami](https://umami.is/) is a fantastic self-hosted alternative to Google Analytics. It gives me the traffic insights I need without harvesting user data. It runs as two containers: the application itself and a Postgres database to store the data.

### The Monitoring Stack

To keep an eye on everything, I run a suite of monitoring tools that follow the standard Prometheus/Grafana model.

* **Prometheus**: The core of my monitoring is [Prometheus](https://prometheus.io/), a powerful time-series database that pulls (scrapes) metrics from various sources.
* **Grafana**: This is the visualization layer. [Grafana](https://grafana.com/) queries the data stored in Prometheus and renders it into useful dashboards, showing the health of everything from the Pi's CPU temperature to container memory usage.
* **The Exporters**: These are the agents that expose the metrics.
  * [Node Exporter](https://github.com/prometheus/node_exporter) provides host-level metrics about the Raspberry Pi itself.
  * [Cadvisor](https://github.com/google/cadvisor) exposes metrics for all the running Docker containers.
  * [Blackbox Exporter](https://github.com/prometheus/blackbox_exporter) probes endpoints (like my blog) to test for uptime and responsiveness.
* **Uptime Kuma**: As a user-friendly frontend, I also run [Uptime Kuma](https://uptimekuma.org/). It provides a simple, clean status page and can send notifications if any service goes down.

### The Secrets and Safeguards

A setup like this has sensitive information like API keys and passwords. It also has a lot of moving parts, making automation and safeguards critical. Here's how I handle the secrets and keep the configuration robust.

* **Unbound**: To enhance privacy, Pi-hole doesn't use a public DNS provider. Instead, it forwards requests to a local [Unbound](https://www.nlnetlabs.nl/projects/unbound/about/) container. Unbound is a validating, recursive DNS resolver that communicates directly with authoritative DNS servers. This means no single third party sees all my DNS traffic.
* **SOPS**: To manage secrets in a Git repository, I use [SOPS](https://getsops.io/) (Secrets OPerationS). It allows me to encrypt a file containing environment variables (`secrets.sops.env`). This encrypted file is safe to commit to the repository, while the plaintext version remains local and ignored by Git.
* **Pre-commit Hooks**: To prevent mistakes like accidentally committing the plaintext secrets file, I use [pre-commit](https://pre-commit.com/) hooks. These are automated scripts that run before a commit is finalized. My hooks check for the presence of the plaintext `.env` file and will block the commit if it's found, preventing secrets from ever leaving my machine.

## The Automation

The final piece of this setup is automation. A manual deployment process is prone to error and time-consuming. The goal here is a completely hands-off '[GitOps](https://www.gitops.tech/)' style workflow, where a `git push` triggers the entire deployment.

The process is handled by a [GitHub Actions](https://github.com/features/actions) workflow defined in `deploy-prod.yaml`. When I push a change to the `main` branch, a self-hosted runner on the Raspberry Pi itself executes the following steps:

{{< codeurl url="https://raw.githubusercontent.com/geeksbsmrt/RaspberryPi/refs/heads/main/.github/workflows/deploy-prod.yaml" lang="yaml" >}}

1. **Checkout Code**: Pulls the latest version of the repository.
2. **Decrypt Secrets**: Uses SOPS and a key stored in GitHub Secrets to decrypt `secrets.sops.env` into the required `.env` file.
3. **Template Configs**: Injects secrets and variables from the `.env` file into configuration templates for services like Prometheus.
4. **Sync Files**: Copies the final configuration files to the Docker directory on the Pi.
5. **Pull Images**: Fetches the latest versions of all Docker images defined in the `docker-compose.yml` file.
6. **Restart Stack**: Runs `docker compose up -d` to apply any changes, restarting containers as needed.
7. **Prune Images**: Cleans up old, unused Docker images to save space.

The result is a fully automated deployment pipeline. A simple `git push` ensures that any change—from updating a service version to changing a dashboard—is applied consistently and reliably, with no manual intervention required.

## The Future

In my next post, I'll cover using Google's [Gemini](https://gemini.google.com) to help me learn and create these services. Let's just say it's been *interesting.*
