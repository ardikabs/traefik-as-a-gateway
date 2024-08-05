# Traefik as a Gateway

This is a sample repository management for Traefik, enabling as Gateway as a Service.

## Overview

The project integrates Traefik with multiple providers, primarily on the `File` provider for routing management and a customizable `Platform` provider for service discovery and upstream policy control. This approach facilitates auto-discovery mechanisms and allows for dynamic traffic manipulation.

### Key Components

- **File Provider**: Manages middlewares and routing configurations directly from Git.
- **_Platform_ Provider**: Handles either [HTTP or TCP service features](https://doc.traefik.io/traefik/routing/services/), such as load balancing, sticky sessions and health checks. If the platform uses Docker, then you can refer to this [docs](https://doc.traefik.io/traefik/routing/providers/docker/#services).

### Workflow

Changes to traffic management are initially made through the `File` provider. These changes are then automatically synchronized across the Traefik fleet using the [git-sync](https://github.com/kubernetes/git-sync) tool, ensuring that the gateway's configuration remains up-to-date with the Git repository. While the available upstreams are synced by utilizing the `Platform` provider, such as but not limited to [Docker](https://doc.traefik.io/traefik/routing/providers/docker/), [Swarm](https://doc.traefik.io/traefik/routing/providers/swarm/), [ECS](https://doc.traefik.io/traefik/routing/providers/ecs/), or [Consul Catalog](https://doc.traefik.io/traefik/routing/providers/consul-catalog/).

### Consideration

The **_Platform_** provider has been effective in automatically creating routers on demand. However, it lacks support for specific scenarios, such as implementing **Weighted Canary** strategies that distribute traffic based on a ratio across different upstreams. This functionality is available in the `File` provider.
Hence, there is a need to have the `File` provider's capabilities for handling such scenarios, then combined with the **_Platform_** provider's ability to manage upstream auto-discovery. This integration will enable the manipulation of traffic for various use cases, including **Weighted Canary**, **A/B Testing**, and more.

## Infrastructure Setup

To achieve this functionality of GitOps, the Traefik entrypoint is customized to operate concurrently with `git-sync`. This is activated through the environment variable `ENABLE_GITSYNC`. Setting `ENABLE_GITSYNC=1` initiates `git-sync`, which clones a specified repository and synchronizes it at intervals set by `GITSYNC_PERIOD`, with a default of 10 seconds (`10s`). Subsequent modifications in the Git repository are automatically updated, triggering the execution hook of [`gateway-hook-configsync`](./bin/gateway-hook-configsync) script. This process populate the changes to the `/etc/traefik` directory, specifically on `/etc/traefik/conf.d`.
Additionally, you need to specify the config path relative to the repo root to indicate which path should be synced into `/etc/traefik`, particularly `/etc/traefik/traefik.yaml` (static config) and `/etc/traefik/conf.d` (dynamic config). For example, to refer to [`envs/staging`](./envs/staging), you can set the environment variable `GATEWAY_CONFIG_PATH=envs/staging`.

### Challenges

The necessity for an execution hook, rather than utilizing `GITSYNC_LINK`, arises because the `File` provider in Traefik, when enables `watch`, [does not effectively handle symbolic links](https://github.com/traefik/traefik/pull/6416). Therefore, in [`gateway-hook-configsync`](./bin/gateway-hook-configsync), `rsync` is employed to accurately synchronize the files to `/etc/traefik`. Consequently, the `File` provider monitors the `/etc/traefik` directory directly, bypassing any issues with symbolic links.
