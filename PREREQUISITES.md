# Host Prerequisites

This document describes how to prepare a host for the Secure AI Devcontainer.

These steps configure host-level software and DevPod settings. They are deliberately not performed automatically by the project.

Once these prerequisites are complete, follow QUICKSTART.md to create and use a project.

Reference environment

The reference Linux setup uses:

* Git
* rootless Podman
* Docker Compose as the Compose implementation used by Podman
* DevPod CLI
* kuju63/devpod-provider-podman v0.4.0
* two DevPod contexts:
  * hardened
  * bunkered

Note: These choices are the project default. Feel free to replace them with your favorite implementation.

⸻

## Install the host packages

In order for the container to work, you need to install the following packages on your host system:

* git
* podman
* docker-compose

The reference configuration uses Docker Compose rather than podman-compose.

Once installed, check the versions:

```bash
git --version
podman --version
```

⸻

## Verify rootless Podman

Podman must run rootless for the current user.

```bash
podman info --format '{{.Host.Security.Rootless}}'

# Expected result: true

# Also, verify that Podman can be used without sudo:

podman ps
```


⸻

## Enable the Podman user socket

On Linux, enable the rootless Podman socket:

```bash
systemctl --user enable --now podman.socket

# Verify:

systemctl --user is-active podman.socket

# Expected result: active

# Verify that the socket exists:

ls -l "$XDG_RUNTIME_DIR/podman/podman.sock"
```

⸻

## Verify Compose support

Podman must have access to a compatible Compose implementation.

```bash
# Verify:

podman compose version
```

The reference Linux setup uses Docker Compose rather than podman-compose.

⸻

## Install DevPod CLI

Install DevPod using the official installation procedure:

* https://devpod.sh/docs/getting-started/install

Follow the Install DevPod CLI instructions.

```bash

# Once installed, Verify:

devpod version

# and:

command -v devpod
```

⸻

## Configure the Hardened context

```bash
# Create the context:

devpod context create hardened

# Select it:

devpod context use hardened

# Install the tested Podman provider:

devpod provider add kuju63/devpod-provider-podman@v0.4.0

# Configure the context:

devpod context set-options \
  --context hardened \
  -o SSH_AGENT_FORWARDING=true \
  -o SSH_ADD_PRIVATE_KEYS=false \
  -o SSH_INJECT_GIT_CREDENTIALS=false \
  -o SSH_INJECT_DOCKER_CREDENTIALS=false \
  -o GIT_SSH_SIGNATURE_FORWARDING=false \
  -o GPG_AGENT_FORWARDING=false

```

Hardened requires: SSH_AGENT_FORWARDING=true

The project does not forward the user’s normal SSH agent directly.

During environment recreation, Hardened creates a dedicated SSH agent containing only explicitly selected identities.

⸻

## Configure the Bunkered context

```bash

# Create the context:

devpod context create bunkered

# Select it:

devpod context use bunkered

# Install the tested Podman provider:

devpod provider add kuju63/devpod-provider-podman@v0.4.0

# Configure the context:

devpod context set-options \
  --context bunkered \
  -o SSH_AGENT_FORWARDING=false \
  -o SSH_ADD_PRIVATE_KEYS=false \
  -o SSH_INJECT_GIT_CREDENTIALS=false \
  -o SSH_INJECT_DOCKER_CREDENTIALS=false \
  -o GIT_SSH_SIGNATURE_FORWARDING=false \
  -o GPG_AGENT_FORWARDING=false

Bunkered requires: SSH_AGENT_FORWARDING=false

No SSH agent is exposed to the Bunkered environment.

⸻

## Verify the DevPod contexts

Verify the SSH forwarding configuration:

```bash

for ctx in hardened bunkered; do
  echo "=== $ctx ==="
  devpod context options --context "$ctx" | grep 'SSH_AGENT_FORWARDING'
done
```

The result should show:

* hardened -> SSH_AGENT_FORWARDING=true
* bunkered -> SSH_AGENT_FORWARDING=false

All automatic credential injection mechanisms should remain disabled in both contexts.

Verify that the Podman provider is available in both contexts:

```bash

for ctx in hardened bunkered; do
  echo "=== $ctx ==="
  devpod context use "$ctx"
  devpod provider list
done
```

The Podman provider must be available in both contexts.

⸻

## Host preparation complete

The host is now ready for the Secure AI Devcontainer.

Once a project has been created from the template, its host configuration can be validated with:

```bash
./scripts/internal/doctor
```

doctor verifies the required host configuration but does not modify it.

Continue with QUICKSTART.md.
