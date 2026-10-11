# Secure AI Devcontainer

A reusable development container designed to run AI coding agents with a
reduced and explicit trust boundary.

The project started from a simple requirement: AI development tools
should be useful inside a project without automatically gaining access
to everything available in the host development environment.

It provides two security modes with different trust levels:

-   **Hardened** --- a practical development environment with controlled
    privilege escalation, selected SSH identities, and optional personal
    dotfiles.
-   **Bunkered** --- a more restrictive environment with no privilege
    escalation, no SSH agent, and no personal dotfiles.

The repository is intended to be used as a GitHub Template Repository.
Each project created from the template becomes independent and can adapt
the security model to its own requirements.

For installation and first use:

-   [PREREQUISITES.md](PREREQUISITES.md) --- prepare the host and
    configure Podman/DevPod.
-   [QUICKSTART.md](QUICKSTART.md) --- create and start a project from
    the template.

## Table of contents

-   [Security model](#security-model)
-   [Hardened vs Bunkered](#hardened-vs-bunkered)
-   [Configuration](#configuration)
-   [Secrets](#secrets)
-   [SSH isolation](#ssh-isolation)
-   [Workspace isolation and
    denylist](#workspace-isolation-and-denylist)
-   [Linux privilege model](#linux-privilege-model)
-   [Repository structure](#repository-structure)
-   [Lifecycle architecture](#lifecycle-architecture)
-   [Runtime scope](#runtime-scope)
-   [Using Docker instead of Podman](#using-docker-instead-of-podman)
-   [Security considerations](#security-considerations)
-   [License](#license)

------------------------------------------------------------------------

## Security model

The devcontainer uses several complementary controls rather than relying
on a single security mechanism.

The main principles are:

-   expose only the project workspace to the container;
-   explicitly mask sensitive files and directories from inside the
    workspace;
-   keep secrets outside the project directory;
-   avoid forwarding the host SSH agent;
-   expose only explicitly selected SSH identities;
-   remove Linux capabilities by default;
-   control privilege escalation according to the selected security
    mode;
-   pass build credentials as build secrets rather than storing them in
    image layers;
-   keep the container configuration itself inaccessible to tools
    running inside the workspace.

These controls are intended to reduce what an AI coding agent can access
or modify if it operates outside its expected scope.

They should be considered defense in depth, not a replacement for the
permission and approval mechanisms provided by the AI tools themselves.

------------------------------------------------------------------------

## Hardened vs Bunkered

  Capability                                    Hardened                        Bunkered
  --------------------------------------------- ------------------------------- -------------------------
  Project workspace                             Yes                             Yes
  mise and development tools                    Yes                             Yes
  zsh                                           Yes                             Yes
  Workspace denylist                            Yes                             Yes
  Personal dotfiles                             Optional                        No
  chezmoi                                       Yes                             No
  SSH agent                                     Dedicated, selected keys only   No
  sudo                                          Password required every time    No privilege escalation
  no-new-privileges                             No                              Yes
  Effective capabilities for development user   None                            None

### Hardened

Hardened mode is intended for normal development when some integration
with the user's environment is required.

It provides:

-   a dedicated SSH agent;
-   explicit selection of the SSH keys exposed to the container;
-   optional personal dotfiles;
-   sudo, protected by a password;
-   no reusable sudo authentication window.

The host SSH agent itself is never forwarded.

Personal dotfiles can be loaded through `chezmoi`.

For a **private GitHub dotfiles repository**, use an SSH repository URL
rather than HTTPS, for example:

``` text
git@github.com:username/dotfiles.git
```

The corresponding public SSH key must be registered with GitHub and have
access to the repository. When recreating the environment, select the
matching private key when prompted for the SSH identities to expose
through the dedicated agent.

The private key itself is not copied into the container. Repository
access is provided through the dedicated SSH agent.

Private dotfiles repositories are supported only in Hardened mode
because Bunkered mode deliberately exposes no SSH agent and installs no
personal dotfiles.

### Bunkered

Bunkered mode minimizes integration with the host environment.

It provides:

-   no SSH agent;
-   no dotfiles;
-   no sudo privilege escalation;
-   `no-new-privileges`;
-   an empty effective capability set for the development user.

This mode is useful when the AI agent should operate with a
significantly smaller trust boundary.

------------------------------------------------------------------------

## Configuration

### `.env.example`

`.env.example` is committed to the repository and documents the
project-level configuration:

``` text
PROJECT_NAME=""
DOTFILES_REPOSITORY=""
```

### `.env`

`scripts/init` generates `.env`.

It contains configuration specific to the project instance:

``` text
PROJECT_NAME="my-project"
DOTFILES_REPOSITORY="git@github.com:username/dotfiles.git"
```

This file contains project configuration, not secrets, and is
intentionally excluded from Git.

`DOTFILES_REPOSITORY` is optional and applies to Hardened mode. For
private repositories, use an SSH URL and ensure that the corresponding
SSH identity is selected for the dedicated agent as described in [SSH
isolation](#ssh-isolation).

`PROJECT_NAME` is also used to isolate project-specific host resources.

For example:

``` text
~/.config/ai-devcontainer/my-project/
```

### `.env.json`

`.env.json` has a different responsibility.

It contains project runtime environment configuration loaded through
mise:

``` toml
[env]
_.file = ".env.json"
```

Do not use `.env.json` for devcontainer configuration.

The distinction is intentional:

``` text
.env       -> devcontainer/template configuration
.env.json  -> project runtime environment
```

------------------------------------------------------------------------

## Secrets

Secrets are deliberately kept outside the project workspace.

Project-specific secrets are stored under:

``` text
~/.config/ai-devcontainer/<PROJECT_NAME>/
```

with restrictive filesystem permissions.

These secrets are persistent across DevPod workspace recreation and
deletion.

For example, the GitHub token used by mise is requested when it is not
already configured for the project and is reused by subsequent
recreations.

Deleting the DevPod workspace does not delete the project's persistent
secrets.

This prevents tools operating only inside the mounted project workspace
from directly reading those files.

Build credentials are passed to Podman using build secrets.

Each image build uses a dedicated temporary build directory containing:

-   the minimal build context;
-   temporary build-secret source files;
-   a private `TMPDIR` used by Podman.

The complete directory is removed automatically when the build finishes,
including when the build fails.

This also ensures that temporary `podman-build-secret-*` files created
by Podman Remote remain confined to the disposable build directory
rather than being left in or next to the project repository.

Build credentials are not intended to become:

-   Dockerfile environment variables;
-   persistent container environment variables;
-   image-layer contents.

The Hardened sudo password is handled separately.

Only its password hash is persisted for the image build. The plaintext
password remains in memory while the recreation workflow is running.

------------------------------------------------------------------------

## SSH isolation

The host SSH agent is not directly forwarded into the devcontainer.

In Hardened mode, the recreation workflow:

1.  discovers valid private SSH keys available on the host;
2.  asks which identities should be exposed;
3.  creates a dedicated `ssh-agent`;
4.  loads only the selected identities;
5.  exposes the dedicated agent socket to DevPod.

The private key files themselves do not need to be mounted into the
container.

The dedicated agent uses project-specific resources under:

``` text
~/.config/ai-devcontainer/<PROJECT_NAME>/
```

This dedicated agent can also provide access to private Git
repositories, including a private dotfiles repository. The repository
must use an SSH URL, and the selected SSH identity must correspond to a
public key authorized by the Git hosting service for that repository.

In Bunkered mode, the dedicated agent is disabled and any existing
project-specific agent is cleaned up. Personal dotfiles are not
installed in this mode.

This avoids implicitly exposing every identity currently loaded into the
user's normal host SSH session.

------------------------------------------------------------------------

## Workspace isolation and denylist

Some files belong to the project repository but should not be accessible
to an AI coding agent running inside the workspace.

The denylist currently includes patterns such as:

``` text
.devcontainer
scripts
**/*.env*
**/*age*
**/*.pem
**/secrets
```

The patterns are resolved on the host before the workspace is exposed to
the container.

Matching directories are masked using empty tmpfs mounts.

Matching files are masked using read-only `/dev/null` mounts.

As a result, a file can remain part of the host repository while its
original content is unavailable from inside the devcontainer.

This is particularly important for:

``` text
.devcontainer/
scripts/
```

The agent can use the environment, but it should not automatically gain
access to the implementation that defines its own security boundary.

The denylist complements --- rather than replaces --- AI CLI permission
rules, allowlists, and approval mechanisms.

### Workspace bind mount

The project workspace is bind-mounted into the container at:

``` text
/workspaces/vscode
```

The Podman mount uses:

``` text
rw,Z
```

rather than relying only on Unix filesystem permissions.

On Linux systems enforcing container security labels, a normal bind
mount can otherwise result in `Permission denied` even when the host
directory and its parents have appropriate Unix permissions.

The `Z` option instructs Podman to relabel the workspace for private use
by the container.

This behavior was required during Linux validation and was also verified
without regression on macOS.

------------------------------------------------------------------------

## Linux privilege model

The generated Compose overlay starts from:

``` yaml
cap_drop:
  - ALL
```

The security mode then restores only what is required for that mode.

In Hardened mode, controlled privilege escalation through sudo is
available.

The sudo configuration requires password authentication for each command
rather than maintaining a reusable authentication window.

In Bunkered mode, privilege escalation is blocked using:

``` text
no-new-privileges
```

The effective capability set of the development user remains empty in
both modes during normal operation.

------------------------------------------------------------------------

## Repository structure

``` text
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
│       ├── doctor
│       ├── build-image
│       ├── devpod-wrapper
│       ├── generate-compose-overlay
│       ├── recreate-environment
│       ├── recreate-workspace
│       ├── setup-secrets
│       └── setup-ssh-agent
├── .env.example
├── mise.toml
├── PREREQUISITES.md
├── QUICKSTART.md
└── README.md
```

The public entry points are:

``` text
scripts/init
scripts/devcontainer
```

The diagnostic command can also be executed directly:

``` bash
./scripts/internal/doctor
```

The remaining scripts under `scripts/internal/` are implementation
details of the lifecycle workflow.

Installation and lifecycle commands are documented in
[PREREQUISITES.md](PREREQUISITES.md) and [QUICKSTART.md](QUICKSTART.md).

------------------------------------------------------------------------

## Lifecycle architecture

Project initialization is intentionally separated from host
configuration:

``` text
./scripts/init
        │
        ▼
      doctor
        │
        ├── verify Podman
        ├── verify rootless runtime
        ├── verify Compose
        ├── verify DevPod
        ├── verify Podman provider
        └── verify security contexts
        │
        ▼
 create project .env
```

`doctor` only validates the host. It does not modify it.

The main recreation workflow is:

``` text
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

``` text
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

This separation is intentional: starting or entering an existing
environment should not silently modify its security configuration.

For the commands and normal workflow, see
[QUICKSTART.md](QUICKSTART.md).

------------------------------------------------------------------------

## Runtime scope

The repository deliberately targets Podman rather than introducing an
additional container-runtime abstraction layer.

The project is validated with:

-   native rootless Podman on Linux;
-   Podman Remote / Podman Machine on macOS.

Host installation and configuration are documented in
[PREREQUISITES.md](PREREQUISITES.md).

------------------------------------------------------------------------

## Using Docker instead of Podman

The repository currently assumes Podman and DevPod contexts configured
with a Podman provider.

This is intentional.

The project targets users who are expected to understand and control
their container runtime, so an additional runtime abstraction layer
would add complexity without improving the security model.

The architecture itself is not Podman-specific.

It can be adapted to Docker by changing the build/runtime integration
and configuring the corresponding DevPod provider.

Such a configuration should be treated as a project-specific adaptation
rather than an automatically supported runtime mode.

------------------------------------------------------------------------

## Security considerations

This project is designed to reduce the trust boundary of AI development
tools, not to claim perfect isolation.

A devcontainer remains a complex environment involving:

-   a container runtime;
-   mounted host resources;
-   DevPod;
-   development tools;
-   AI agents;
-   network access;
-   external services.

The controls implemented here should therefore be considered part of a
defense-in-depth strategy.

Before using the environment with sensitive repositories, review:

-   the workspace mounts;
-   the denylist;
-   SSH identities;
-   container capabilities;
-   secrets handling;
-   AI tool permissions;
-   network access;
-   any additional host resources exposed by your DevPod configuration.

Host-level dependencies such as Podman, DevPod, Compose implementations
and third-party DevPod providers remain part of the host trust boundary.

The repository validates some security-sensitive configuration through
`doctor`, but this does not replace reviewing and maintaining those
dependencies.

The appropriate security mode depends on how much access the development
workflow actually requires.

------------------------------------------------------------------------

## License

This project is licensed under the MIT License.
