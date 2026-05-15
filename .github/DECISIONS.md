# Architectural Decisions

Intentional choices that look like issues in automated review but are deliberate for this homelab setup.

## Floating image tags (`latest`)

`ghcr.io/majowka/arc-runner:latest`, `docker:dind`, `bitnami/kubectl:latest` are intentionally unpinned.

- `arc-runner` is our own image, rebuilt every Monday 04:00 UTC and on every Dockerfile change (see [Majowka/arc-runner](https://github.com/Majowka/arc-runner)). Pinning a digest would require a separate automation to bump it on every rebuild — net complexity gain with no real benefit in a homelab.
- `docker:dind` and `bitnami/kubectl` follow the same rationale: we accept minor version drift in exchange for always getting security patches without manual maintenance.

If this repo ever moves to a production workload, pin images and add Renovate/Dependabot.

## HTTP registry cache in `insecure-registries`

`mirror.gcr.io`, `mcr.microsoft.com`, and `registry-1.docker.io` appear in Docker's `insecure-registries`. This does **not** mean we pull from the public internet over plain HTTP.

The dind sidecar manipulates `/etc/hosts` to redirect those hostnames to in-cluster HTTP cache services (`registry-cache-*.<zone>.arc-runners.svc.cluster.local:5000`). TLS is not available on those services by design (pull-through cache, emptyDir-backed). The `insecure-registries` entry tells Docker to accept HTTP responses from those hostnames after the redirect.

Traffic never leaves the cluster for cached images. If the cache is unavailable, the health-check fallback in the dind init script skips the `/etc/hosts` entry and Docker falls back to the real public registry over HTTPS.
