# Ruby Project - InvisiRisk BAF Example Setup

This guide explains how to integrate the InvisiRisk BAF into your AWS CodeBuild pipeline using the BuildKit frontend approach. **No changes to your `Dockerfile` are required.** This setup assumes an Ubuntu runner with Docker pre-installed.

## Prerequisites

Ensure the `API_URL` and `APP_TOKEN` environment variables are set in your build environment before applying these changes.

---

## Step 1: Modify `buildspec.yml`

Update your `buildspec.yml` to include the BAF startup and cleanup steps, and update your `docker build` command.

### `pre_build` phase - start the BAF:

```yaml
pre_build:
  commands:
    - echo "InvisiRisk startup script..."
    - curl $API_URL/pse/bitbucket-setup/pse_startup | bash # Download and execute the BAF setup script.
    - . /etc/profile.d/pse-proxy.sh # Source the environment variables set by the setup script.
```

### `build` phase - updated `docker build` command:

```yaml
build:
  commands:
    - echo "Docker build with InvisiRisk BAF..."
    - DOCKER_BUILDKIT=1 docker build --no-cache --platform linux/amd64 -t $IMAGE_NAME:$CODEBUILD_BUILD_NUMBER --build-arg BUILDKIT_SYNTAX=public.ecr.aws/w3c0c0n7/invisirisk/baf-buildkit:latest --secret id=pse-ca,src=/etc/ssl/certs/pse.pem --build-arg PSE_PROXY=http://${PSE_PROXY_IP}:3128 .
```

### `post_build` phase - run the cleanup script:

```yaml
post_build:
  commands:
    - echo "Build complete!"
    - bash /tmp/pse_cleanup/cleanup.sh # Sends build data to the InvisiRisk portal.
```

### Full `buildspec.yml` example:

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo "InvisiRisk startup script..."
      - curl $API_URL/pse/bitbucket-setup/pse_startup | bash
      - . /etc/profile.d/pse-proxy.sh

  build:
    commands:
      - echo "Docker build with InvisiRisk BAF..."
      - DOCKER_BUILDKIT=1 docker build --no-cache --platform linux/amd64 -t $IMAGE_NAME:$CODEBUILD_BUILD_NUMBER --build-arg BUILDKIT_SYNTAX=public.ecr.aws/w3c0c0n7/invisirisk/baf-buildkit:latest --secret id=pse-ca,src=/etc/ssl/certs/pse.pem --build-arg PSE_PROXY=http://${PSE_PROXY_IP}:3128 .

  post_build:
    commands:
      - echo "Build complete!"
      - bash /tmp/pse_cleanup/cleanup.sh
```

---

## Step 2: `docker build` Command Changes

The BAF integration requires the following additions to your `docker build` command. These are already reflected in the `buildspec.yml` above - refer to this section if you need to apply the changes to a different CI setup or a standalone Docker build.

| Argument | What it does |
|---|---|
| `DOCKER_BUILDKIT=1` | Enables BuildKit mode, required for the custom frontend and secrets support. Add this as a prefix if not already set. |
| `--build-arg BUILDKIT_SYNTAX=public.ecr.aws/w3c0c0n7/invisirisk/baf-buildkit:latest` | Tells Docker BuildKit to use the InvisiRisk custom frontend, which transparently routes build-time traffic through the BAF. |
| `--secret id=pse-ca,src=/etc/ssl/certs/pse.pem` | Passes the PSE CA certificate to the build without embedding it in the final image. |
| `--build-arg PSE_PROXY=http://${PSE_PROXY_IP}:3128` | Tells the BAF frontend which proxy endpoint to use. `PSE_PROXY_IP` is set by the startup script. |

```sh
DOCKER_BUILDKIT=1 docker build \
  --build-arg BUILDKIT_SYNTAX=public.ecr.aws/w3c0c0n7/invisirisk/baf-buildkit:latest \
  --secret id=pse-ca,src=/etc/ssl/certs/pse.pem \
  --build-arg PSE_PROXY=http://${PSE_PROXY_IP}:3128 \
  [your existing flags] .
```

---

## Required Environment Variables

The following environment variables must be set in your build environment:

| Variable     | Description                               |
|--------------|-------------------------------------------|
| `API_URL`    | https://app.invisirisk.com                |
| `APP_TOKEN`  | APP token received from InvisiRisk portal |

---

## Notes

- The BAF startup must complete before the `build` phase so that all network traffic during dependency installation is routed correctly.
- `PSE_PROXY_IP` is set automatically by the PSE startup script and sourced via `/etc/profile.d/pse-proxy.sh`.
- The cleanup script in `post_build` should always run, even if the build fails.

