# Secure AI Devcontainer

A reusable development container designed to run AI coding agents with a reduced and explicit trust boundary.

The project started from a simple requirement: AI development tools should be useful inside a project without automatically gaining access to everything available in the host development environment.

It provides two security modes with different trust levels:

- **Hardened** — a practical development environment with controlled privilege escalation, selected SSH identities, and personal dotfiles.
- **Bunkered** — a more restrictive environment with no privilege escalation, no SSH agent, and no personal dotfiles.

The repository is intended to be used as a **GitHub Template Repository**. Each project created from the template becomes independent and can adapt the security model to its own requirements.

## Table of contents

- [Security model](#security-model)
- [Hardened vs Bunkered](#hardened-vs-bunkered)
- [Prerequisites](#prerequisites)
- [Create a project](#create-a-project)
- [Daily workflow](#daily-workflow)
- [Configuration](#configuration)
- [Secrets](#secrets)
- [SSH isolation](#ssh-isolation)
- [Workspace isolation and denylist](#workspace-isolation-and-denylist)
- [Linux privilege model](#linux-privilege-model)
- [Repository structure](#repository-structure)
- [Lifecycle architecture](#lifecycle-architecture)
- [Using Docker instead of Podman](#using-docker-instead-of-podman)
- [Security considerations](#security-considerations)
- [License](#license)

---

## Security model

The devcontainer uses several complementary controls rather than relying on a single security mechanism.

The main principles are:

- expose only the project workspace to the container;
- explicitly mask sensitive files and directories from inside the workspace;
- keep secrets outside the project directory;
- avoid forwarding the host SSH agent;
- expose only explicitly selected SSH identities;
- remove Linux capabilities by default;
- control privilege escalation according to the selected security mode;
- pass build credentials as build secrets rather than storing them in image layers;
- keep the container configuration itself inaccessible to tools running inside the workspace.

These controls are intended to reduce what an AI coding agent can access or modify if it operates outside its expected scope.

They should be considered **defense in depth**, not a replacement for the permission and approval mechanisms provided by the AI tools themselves.

---

## Hardened vs Bunkered

| Capability | Hardened | Bunkered |
|---|---|---|
| Project workspace | Yes | Yes |
| `mise` and development tools | Yes | Yes |
| `zsh` | Yes | Yes |
| Workspace denylist | Yes | Yes |
| Personal dotfiles | Optional | No |
| `chezmoi` | Yes | No |
| SSH agent | Dedicated, selected keys only | No |
| `sudo` | Password required every time | No privilege escalation |
| `no-new-privileges` | No | Yes |
| Effective capabilities for development user | None | None |

### Hardened

Hardened mode is intended for normal development when some integration with the user's environment is required.

It provides:

- a dedicated SSH agent;
- explicit selection of the SSH keys exposed to the container;
- optional personal dotfiles;
- `sudo`, protected by a password;
- no reusable sudo authentication window.

The host SSH agent itself is never forwarded.

### Bunkered

Bunkered mode minimizes integration with the host environment.

It provides:

- no SSH agent;
- no dotfiles;
- no `sudo` privilege escalation;
- `no-new-privileges`;
- an empty effective capability set for the development user.

This mode is useful when the AI agent should operate with a significantly smaller trust boundary.

---

## Prerequisites

The current implementation uses:

- Git
- Podman
- DevPod
- a GitHub token used by `mise` while installing development tools

DevPod must be configured with the **Podman provider**.

The repository deliberately targets Podman rather than introducing an additional container-runtime abstraction layer.

The same architecture can be adapted to DevPod with Docker if required, but that configuration is not maintained by this repository.

For Hardened mode, an SSH-accessible dotfiles repository is optional.

---

## Create a project

This repository is designed to be configured as a GitHub Template Repository.

### 1. Create a repository from the template

On GitHub, select:

```text
Use this template
```

and create a new repository for your project.

For example:

```text
my-project
```

### 2. Clone the new repository

```bash
git clone <your-new-repository>
cd my-project
```

### 3. Initialize the local configuration

```bash
./scripts/init
```

The initialization script:

- derives the project name from the repository directory;
- asks for an optional dotfiles repository;
- creates the local `.env` configuration;
- refuses to overwrite an existing `.env`.

Example:

```bash
PROJECT_NAME="my-project"
DOTFILES_REPOSITORY="git@github.com:username/dotfiles.git"
```

The generated `.env` is local to the project and is not committed to Git.

### 4. Create the devcontainer

```bash
./scripts/devcontainer up --recreate
```

This is the configuration path for the environment.

It will:

1. ask for the security mode;
2. configure the required secrets;
3. configure the dedicated SSH agent when using Hardened mode;
4. build the container image;
5. generate the security-specific Compose overlay;
6. recreate the DevPod workspace.

On the first recreation, the environment will also request the GitHub token required by the build when it is not already configured.

In Hardened mode, it will additionally request:

- the sudo password for this recreation;
- the SSH identities to expose to the dedicated agent.

Bunkered mode does not configure a sudo password or SSH agent.

---

## Daily workflow

The lifecycle commands intentionally distinguish between **using an existing environment** and **reconfiguring it**.

### Start an existing environment

```bash
./scripts/devcontainer up
```

This starts the existing DevPod workspace without changing its configuration.

It does **not**:

- select a security mode;
- rebuild the image;
- regenerate the Compose overlay;
- request a sudo password;
- request SSH keys.

Use this when the existing container has simply been stopped.

### Connect to the environment

```bash
./scripts/devcontainer ssh
```

This connects to the running container and starts the session in:

```text
/workspaces/vscode
```

Connecting never modifies the environment configuration.

### Recreate the environment

```bash
./scripts/devcontainer up --recreate
```

Use this when the environment itself must be rebuilt or reconfigured.

This is the only normal lifecycle command that performs the complete configuration workflow.

### Delete the environment

```bash
./scripts/devcontainer delete
```

This deletes the DevPod workspace and cleans up the dedicated SSH agent associated with the project.

---

## Configuration

### `.env.example`

`.env.example` is committed to the repository and documents the project-level configuration:

```bash
PROJECT_NAME=""
DOTFILES_REPOSITORY=""
```

### `.env`

`scripts/init` generates `.env`.

It contains configuration specific to the project instance:

```bash
PROJECT_NAME="my-project"
DOTFILES_REPOSITORY="git@github.com:username/dotfiles.git"
```

This file is intentionally excluded from Git.

`PROJECT_NAME` is also used to isolate project-specific host resources.

For example:

```text
~/.config/ai-devcontainer/my-project/
```

### `.env.json`

`.env.json` has a different responsibility.

It contains project runtime environment configuration loaded through `mise`:

```toml
[env]
_.file = ".env.json"
```

Do not use `.env.json` for devcontainer configuration.

The distinction is intentional:

```text
.env       -> devcontainer/template configuration
.env.json  -> project runtime environment
```

---

## Secrets

Secrets are deliberately kept outside the project workspace.

Project-specific secrets are stored under:

```text
~/.config/ai-devcontainer/<PROJECT_NAME>/
```

with restrictive filesystem permissions.

These secrets are persistent across DevPod workspace recreation and deletion.

For example, the GitHub token used by `mise` is requested when it is not already configured for the project and is reused by subsequent recreations.

Deleting the DevPod workspace does not delete the project's persistent secrets.

This prevents tools operating only inside the mounted project workspace from directly reading those files.

Build credentials are passed to Podman using build secrets.

Each image build uses a dedicated temporary build directory.

The directory contains:

- the minimal build context;
- temporary build-secret source files;
- a private `TMPDIR` used by Podman.

The complete directory is removed automatically when the build finishes, including when the build fails.

This also ensures that temporary `podman-build-secret-*` files created by Podman Remote remain confined to the disposable build directory rather than being left in or next to the project repository.

They are not intended to become:

- Dockerfile environment variables;
- persistent container environment variables;
- image-layer contents.

The Hardened sudo password is handled separately.

Only its password hash is persisted for the image build. The plaintext password remains in memory while the recreation workflow is running.

---

## SSH isolation

The host SSH agent is not directly forwarded into the devcontainer.

In Hardened mode, the recreation workflow:

1. discovers valid private SSH keys available on the host;
2. asks which identities should be exposed;
3. creates a dedicated `ssh-agent`;
4. loads only the selected identities;
5. exposes the dedicated agent socket to DevPod.

The private key files themselves do not need to be mounted into the container.

The dedicated agent uses project-specific resources under:

```text
~/.config/ai-devcontainer/<PROJECT_NAME>/
```

In Bunkered mode, the dedicated agent is disabled and any existing project-specific agent is cleaned up.

This avoids implicitly exposing every identity currently loaded into the user's normal host SSH session.

---

## Workspace isolation and denylist

Some files belong to the project repository but should not be accessible to an AI coding agent running inside the workspace.

The denylist currently includes patterns such as:

```text
.devcontainer
scripts
**/*.env*
**/*age*
**/*.pem
**/secrets
```

The patterns are resolved on the host before the workspace is exposed to the container.

Matching directories are masked using empty `tmpfs` mounts.

Matching files are masked using read-only `/dev/null` mounts.

As a result, a file can remain part of the host repository while its original content is unavailable from inside the devcontainer.

This is particularly important for:

```text
.devcontainer/
scripts/
```

The agent can use the environment, but it should not automatically gain access to the implementation that defines its own security boundary.

The denylist complements — rather than replaces — AI CLI permission rules, allowlists, and approval mechanisms.

---

## Linux privilege model

The generated Compose overlay starts from:

```yaml
cap_drop:
  - ALL
```

The security mode then restores only what is required for that mode.

In Hardened mode, controlled privilege escalation through `sudo` is available.

The sudo configuration requires password authentication for each command rather than maintaining a reusable authentication window.

In Bunkered mode, privilege escalation is blocked using:

```text
no-new-privileges
```

The effective capability set of the development user remains empty in both modes during normal operation.

---

## Repository structure

```text
.
├── .devcontainer/
│   ├── Dockerfile
│   ├── devcontainer.json
│   ├── docker-compose.yaml
│   └── ...
├── scripts/
│   ├── devcontainer
│   ├── init
│   └── internal/
│       ├── build-image
│       ├── devpod-wrapper
│       ├── generate-compose-overlay
│       ├── recreate-environment
│       ├── recreate-workspace
│       ├── setup-secrets
│       └── setup-ssh-agent
├── .env.example
├── mise.toml
└── README.md
```

The two scripts intended for normal direct use are:

```text
scripts/init
scripts/devcontainer
```

The scripts under:

```text
scripts/internal/
```

are implementation details of the lifecycle workflow.

---

## Lifecycle architecture

The main recreation workflow is:

```text
./scripts/devcontainer up --recreate
        │
        ▼
select security mode
        │
        ▼
recreate-environment
        │
        ├── setup-secrets
        │
        ├── setup-ssh-agent
        │
        ├── build-image
        │
        ├── generate-compose-overlay
        │
        └── recreate-workspace
                    │
                    ▼
                  DevPod
```

Regular operations deliberately bypass this configuration workflow:

```text
./scripts/devcontainer up
        │
        └── start existing DevPod workspace

./scripts/devcontainer ssh
        │
        └── connect to existing workspace

./scripts/devcontainer delete
        │
        └── delete workspace + SSH agent cleanup
```

This separation is intentional: starting or entering an existing environment should not silently modify its security configuration.

---

## Using Docker instead of Podman

The repository currently assumes Podman and a DevPod context configured with the Podman provider.

This is intentional.

The project targets users who are expected to understand and control their container runtime, so an additional runtime abstraction layer would add complexity without improving the security model.

The architecture itself is not Podman-specific.

It can be adapted to Docker by changing the build/runtime integration and configuring the corresponding DevPod provider.

Such a configuration should be treated as a project-specific adaptation rather than an automatically supported runtime mode.

---

## Security considerations

This project is designed to **reduce the trust boundary of AI development tools**, not to claim perfect isolation.

A devcontainer remains a complex environment involving:

- a container runtime;
- mounted host resources;
- DevPod;
- development tools;
- AI agents;
- network access;
- external services.

The controls implemented here should therefore be considered part of a defense-in-depth strategy.

Before using the environment with sensitive repositories, review:

- the workspace mounts;
- the denylist;
- SSH identities;
- container capabilities;
- secrets handling;
- AI tool permissions;
- network access;
- any additional host resources exposed by your DevPod configuration.

The appropriate security mode depends on how much access the development workflow actually requires.

## License

This project is licensed under the [MIT License](LICENSE).
