Secure AI Devcontainer

A reusable development container designed to run AI coding agents with a reduced and explicit trust boundary.

The project started from a simple requirement: AI development tools should be useful inside a project without automatically gaining access to everything available in the host development environment.

It provides two security modes with different trust levels:

* Hardened — a practical development environment with controlled privilege escalation, selected SSH identities, and optional personal dotfiles.
* Bunkered — a more restrictive environment with no privilege escalation, no SSH agent, and no personal dotfiles.

The repository is intended to be used as a GitHub Template Repository. Each project created from the template becomes independent and can adapt the security model to its own requirements.

Table of contents

* Security model
* Hardened vs Bunkered
* Prerequisites
* Create a project
* Daily workflow
* Configuration
* Secrets
* SSH isolation
* Workspace isolation and denylist
* Linux privilege model
* Repository structure
* Lifecycle architecture
* Using Docker instead of Podman
* Security considerations
* License

⸻

Security model

The devcontainer uses several complementary controls rather than relying on a single security mechanism.

The main principles are:

* expose only the project workspace to the container;
* explicitly mask sensitive files and directories from inside the workspace;
* keep secrets outside the project directory;
* avoid forwarding the host SSH agent;
* expose only explicitly selected SSH identities;
* remove Linux capabilities by default;
* control privilege escalation according to the selected security mode;
* pass build credentials as build secrets rather than storing them in image layers;
* keep the container configuration itself inaccessible to tools running inside the workspace.

These controls are intended to reduce what an AI coding agent can access or modify if it operates outside its expected scope.

They should be considered defense in depth, not a replacement for the permission and approval mechanisms provided by the AI tools themselves.

⸻

Hardened vs Bunkered

Capability	Hardened	Bunkered
Project workspace	Yes	Yes
mise and development tools	Yes	Yes
zsh	Yes	Yes
Workspace denylist	Yes	Yes
Personal dotfiles	Optional	No
chezmoi	Yes	No
SSH agent	Dedicated, selected keys only	No
sudo	Password required every time	No privilege escalation
no-new-privileges	No	Yes
Effective capabilities for development user	None	None

Hardened

Hardened mode is intended for normal development when some integration with the user’s environment is required.

It provides:

* a dedicated SSH agent;
* explicit selection of the SSH keys exposed to the container;
* optional personal dotfiles;
* sudo, protected by a password;
* no reusable sudo authentication window.

The host SSH agent itself is never forwarded.

Bunkered

Bunkered mode minimizes integration with the host environment.

It provides:

* no SSH agent;
* no dotfiles;
* no sudo privilege escalation;
* no-new-privileges;
* an empty effective capability set for the development user.

This mode is useful when the AI agent should operate with a significantly smaller trust boundary.

⸻

Prerequisites

The project deliberately does not install or configure host-level dependencies.

The host environment must be prepared before running scripts/init. The project provides a diagnostic command that verifies the expected configuration without modifying it:

./scripts/internal/doctor

The current implementation requires:

* Git;
* Podman;
* rootless Podman;
* working Podman Compose support;
* DevPod;
* a DevPod Podman provider;
* two DevPod contexts named hardened and bunkered;
* a GitHub token used by mise while installing development tools.

Podman

Podman must be installed and usable by the current user in rootless mode.

Verify:

podman info
podman ps

On Linux, the Podman user socket may need to be enabled:

systemctl --user enable --now podman.socket

This is host configuration and is deliberately not performed automatically by this repository.

Compose support

The project requires working Compose support through Podman.

Verify:

podman compose version

Depending on the host environment, this may require installing a compatible Compose provider such as Docker Compose or Podman Compose.

The repository does not install or select a Compose implementation automatically.

DevPod

DevPod must be installed and available:

devpod version

DevPod Podman provider

DevPod must have a Podman provider installed.

The project has been tested with the community provider:

kuju63/devpod-provider-podman

and specifically with:

v0.4.0

A reproducible installation using the tested version can be performed with:

devpod provider add kuju63/devpod-provider-podman@v0.4.0

This provider is a third-party dependency and is not installed, maintained, or updated by this repository.

The command above documents the configuration used to validate this project. Users remain free to install or configure a compatible Podman provider differently.

Verify that the provider is available:

devpod provider list

DevPod contexts

The project requires two independent DevPod contexts:

hardened
bunkered

Create them if necessary:

devpod context create hardened
devpod context create bunkered

Configure the Hardened context:

devpod context set-options \
  --context hardened \
  -o SSH_AGENT_FORWARDING=true \
  -o SSH_ADD_PRIVATE_KEYS=false \
  -o SSH_INJECT_GIT_CREDENTIALS=false \
  -o SSH_INJECT_DOCKER_CREDENTIALS=false \
  -o GIT_SSH_SIGNATURE_FORWARDING=false \
  -o GPG_AGENT_FORWARDING=false

Configure the Bunkered context:

devpod context set-options \
  --context bunkered \
  -o SSH_AGENT_FORWARDING=false \
  -o SSH_ADD_PRIVATE_KEYS=false \
  -o SSH_INJECT_GIT_CREDENTIALS=false \
  -o SSH_INJECT_DOCKER_CREDENTIALS=false \
  -o GIT_SSH_SIGNATURE_FORWARDING=false \
  -o GPG_AGENT_FORWARDING=false

The important difference is:

Hardened  -> SSH_AGENT_FORWARDING=true
Bunkered  -> SSH_AGENT_FORWARDING=false

All automatic credential injection mechanisms are explicitly disabled in both contexts.

The project manages its own dedicated SSH agent in Hardened mode rather than forwarding the user’s normal host SSH agent.

The Podman provider must also be available to both contexts according to the DevPod/provider configuration used on the host.

Verify the host configuration

Run:

./scripts/internal/doctor

The diagnostic verifies:

* Podman is installed;
* Podman is rootless;
* Podman is usable;
* Podman Compose support is available;
* DevPod is installed;
* the Podman provider is available;
* the hardened context exists;
* the bunkered context exists;
* both contexts have the expected security-sensitive options.

doctor is intentionally read-only.

It does not:

* install software;
* create contexts;
* modify context options;
* configure providers;
* start services;
* modify Podman configuration.

If a prerequisite is missing, it reports the problem and exits with a non-zero status.

Platform notes

The project is tested with:

* native rootless Podman on Linux;
* Podman Remote / Podman Machine on macOS.

The workspace bind mount uses:

rw,Z

The Z option allows Podman to apply the security relabel required by hosts where container security labeling would otherwise prevent access to the workspace.

This was required during Linux validation and was also verified without regression on macOS.

Runtime scope

The repository deliberately targets Podman rather than introducing an additional container-runtime abstraction layer.

The same architecture can be adapted to DevPod with Docker if required, but that configuration is not maintained by this repository.

For Hardened mode, an SSH-accessible dotfiles repository is optional.

⸻

Create a project

This repository is designed to be configured as a GitHub Template Repository.

1. Create a repository from the template

On GitHub, select:

Use this template

and create a new repository for your project.

For example:

my-project

2. Clone the new repository

git clone <your-new-repository>
cd my-project

3. Initialize the local configuration

./scripts/init

The initialization script first runs the host prerequisite checks performed by doctor.

If a prerequisite is missing, initialization stops before creating any project configuration.

init does not install or configure Podman, DevPod, providers, contexts, or host services.

Once the prerequisite checks succeed, the initialization script:

* derives the project name from the repository directory;
* asks for an optional dotfiles repository;
* creates the local .env configuration;
* refuses to overwrite an existing .env.

Example:

PROJECT_NAME="my-project"
DOTFILES_REPOSITORY="git@github.com:username/dotfiles.git"

The generated .env is local to the project and is not committed to Git.

4. Create the devcontainer

./scripts/devcontainer up --recreate

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

* the sudo password for this recreation;
* the SSH identities to expose to the dedicated agent.

Bunkered mode does not configure a sudo password or SSH agent.

⸻

Daily workflow

The lifecycle commands intentionally distinguish between using an existing environment and reconfiguring it.

Start an existing environment

./scripts/devcontainer up

This starts the existing DevPod workspace without changing its configuration.

It does not:

* select a security mode;
* rebuild the image;
* regenerate the Compose overlay;
* request a sudo password;
* request SSH keys.

Use this when the existing container has simply been stopped.

Connect to the environment

./scripts/devcontainer ssh

This connects to the running container and starts the session in:

/workspaces/vscode

Connecting never modifies the environment configuration.

Recreate the environment

./scripts/devcontainer up --recreate

Use this when the environment itself must be rebuilt or reconfigured.

This is the only normal lifecycle command that performs the complete configuration workflow.

Delete the environment

./scripts/devcontainer delete

This deletes the DevPod workspace and cleans up the dedicated SSH agent associated with the project.

⸻

Configuration

.env.example

.env.example is committed to the repository and documents the project-level configuration:

PROJECT_NAME=""
DOTFILES_REPOSITORY=""

.env

scripts/init generates .env.

It contains configuration specific to the project instance:

PROJECT_NAME="my-project"
DOTFILES_REPOSITORY="git@github.com:username/dotfiles.git"

This file is intentionally excluded from Git.

PROJECT_NAME is also used to isolate project-specific host resources.

For example:

~/.config/ai-devcontainer/my-project/

.env.json

.env.json has a different responsibility.

It contains project runtime environment configuration loaded through mise:

[env]
_.file = ".env.json"

Do not use .env.json for devcontainer configuration.

The distinction is intentional:

.env       -> devcontainer/template configuration
.env.json  -> project runtime environment

⸻

Secrets

Secrets are deliberately kept outside the project workspace.

Project-specific secrets are stored under:

~/.config/ai-devcontainer/<PROJECT_NAME>/

with restrictive filesystem permissions.

These secrets are persistent across DevPod workspace recreation and deletion.

For example, the GitHub token used by mise is requested when it is not already configured for the project and is reused by subsequent recreations.

Deleting the DevPod workspace does not delete the project’s persistent secrets.

This prevents tools operating only inside the mounted project workspace from directly reading those files.

Build credentials are passed to Podman using build secrets.

Each image build uses a dedicated temporary build directory.

The directory contains:

* the minimal build context;
* temporary build-secret source files;
* a private TMPDIR used by Podman.

The complete directory is removed automatically when the build finishes, including when the build fails.

This also ensures that temporary podman-build-secret-* files created by Podman Remote remain confined to the disposable build directory rather than being left in or next to the project repository.

They are not intended to become:

* Dockerfile environment variables;
* persistent container environment variables;
* image-layer contents.

The Hardened sudo password is handled separately.

Only its password hash is persisted for the image build. The plaintext password remains in memory while the recreation workflow is running.

⸻

SSH isolation

The host SSH agent is not directly forwarded into the devcontainer.

In Hardened mode, the recreation workflow:

1. discovers valid private SSH keys available on the host;
2. asks which identities should be exposed;
3. creates a dedicated ssh-agent;
4. loads only the selected identities;
5. exposes the dedicated agent socket to DevPod.

The private key files themselves do not need to be mounted into the container.

The dedicated agent uses project-specific resources under:

~/.config/ai-devcontainer/<PROJECT_NAME>/

In Bunkered mode, the dedicated agent is disabled and any existing project-specific agent is cleaned up.

This avoids implicitly exposing every identity currently loaded into the user’s normal host SSH session.

⸻

Workspace isolation and denylist

Some files belong to the project repository but should not be accessible to an AI coding agent running inside the workspace.

The denylist currently includes patterns such as:

.devcontainer
scripts
**/*.env*
**/*age*
**/*.pem
**/secrets

The patterns are resolved on the host before the workspace is exposed to the container.

Matching directories are masked using empty tmpfs mounts.

Matching files are masked using read-only /dev/null mounts.

As a result, a file can remain part of the host repository while its original content is unavailable from inside the devcontainer.

This is particularly important for:

.devcontainer/
scripts/

The agent can use the environment, but it should not automatically gain access to the implementation that defines its own security boundary.

The denylist complements — rather than replaces — AI CLI permission rules, allowlists, and approval mechanisms.

Workspace bind mount

The project workspace is bind-mounted into the container at:

/workspaces/vscode

The Podman mount uses:

rw,Z

rather than relying only on Unix filesystem permissions.

On Linux systems enforcing container security labels, a normal bind mount can otherwise result in Permission denied even when the host directory and its parents have appropriate Unix permissions.

The Z option instructs Podman to relabel the workspace for private use by the container.

⸻

Linux privilege model

The generated Compose overlay starts from:

cap_drop:
  - ALL

The security mode then restores only what is required for that mode.

In Hardened mode, controlled privilege escalation through sudo is available.

The sudo configuration requires password authentication for each command rather than maintaining a reusable authentication window.

In Bunkered mode, privilege escalation is blocked using:

no-new-privileges

The effective capability set of the development user remains empty in both modes during normal operation.

⸻

Repository structure

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
└── README.md

The two scripts intended for normal lifecycle use are:

scripts/init
scripts/devcontainer

The diagnostic command can also be executed directly:

./scripts/internal/doctor

The remaining scripts under:

scripts/internal/

are implementation details of the lifecycle workflow.

⸻

Lifecycle architecture

Project initialization is intentionally separated from host configuration:

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

doctor only validates the host. It does not modify it.

The main recreation workflow is:

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

Regular operations deliberately bypass this configuration workflow:

./scripts/devcontainer up
        │
        └── start existing DevPod workspace
./scripts/devcontainer ssh
        │
        └── connect to existing workspace
./scripts/devcontainer delete
        │
        └── delete workspace + SSH agent cleanup

This separation is intentional: starting or entering an existing environment should not silently modify its security configuration.

⸻

Using Docker instead of Podman

The repository currently assumes Podman and DevPod contexts configured with a Podman provider.

This is intentional.

The project targets users who are expected to understand and control their container runtime, so an additional runtime abstraction layer would add complexity without improving the security model.

The architecture itself is not Podman-specific.

It can be adapted to Docker by changing the build/runtime integration and configuring the corresponding DevPod provider.

Such a configuration should be treated as a project-specific adaptation rather than an automatically supported runtime mode.

⸻

Security considerations

This project is designed to reduce the trust boundary of AI development tools, not to claim perfect isolation.

A devcontainer remains a complex environment involving:

* a container runtime;
* mounted host resources;
* DevPod;
* development tools;
* AI agents;
* network access;
* external services.

The controls implemented here should therefore be considered part of a defense-in-depth strategy.

Before using the environment with sensitive repositories, review:

* the workspace mounts;
* the denylist;
* SSH identities;
* container capabilities;
* secrets handling;
* AI tool permissions;
* network access;
* any additional host resources exposed by your DevPod configuration.

Host-level dependencies such as Podman, DevPod, Compose implementations and third-party DevPod providers remain part of the host trust boundary.

The repository validates some security-sensitive configuration through doctor, but this does not replace reviewing and maintaining those dependencies.

The appropriate security mode depends on how much access the development workflow actually requires.

License

This project is licensed under the MIT License.
