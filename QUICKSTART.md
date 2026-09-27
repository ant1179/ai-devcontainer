Quick Start

This guide provides the shortest path to creating a project from the Secure AI Devcontainer template.

For details about the security model, configuration and architecture, see the README.

1. Install the prerequisites

The host requires:

* Git
* Podman running rootless
* Podman Compose support
* DevPod
* the DevPod Podman provider

See Prerequisites for details.

Verify Podman:

podman info
podman ps
podman compose version

On Linux, if the Podman user socket is required but not running:

systemctl --user enable --now podman.socket

2. Install the DevPod Podman provider

The project is tested with kuju63/devpod-provider-podman v0.4.0:

devpod provider add kuju63/devpod-provider-podman@v0.4.0

Verify:

devpod provider list

See DevPod Podman provider.

3. Create the DevPod contexts

Create the two required contexts:

devpod context create hardened
devpod context create bunkered

Configure Hardened:

devpod context set-options \
  --context hardened \
  -o SSH_AGENT_FORWARDING=true \
  -o SSH_ADD_PRIVATE_KEYS=false \
  -o SSH_INJECT_GIT_CREDENTIALS=false \
  -o SSH_INJECT_DOCKER_CREDENTIALS=false \
  -o GIT_SSH_SIGNATURE_FORWARDING=false \
  -o GPG_AGENT_FORWARDING=false

Configure Bunkered:

devpod context set-options \
  --context bunkered \
  -o SSH_AGENT_FORWARDING=false \
  -o SSH_ADD_PRIVATE_KEYS=false \
  -o SSH_INJECT_GIT_CREDENTIALS=false \
  -o SSH_INJECT_DOCKER_CREDENTIALS=false \
  -o GIT_SSH_SIGNATURE_FORWARDING=false \
  -o GPG_AGENT_FORWARDING=false

See DevPod contexts for details.

4. Create a project from the template

On GitHub:

Use this template

Create your new repository, then clone it:

git clone <your-new-repository>
cd <your-project>

See Create a project.

5. Check the host

Run:

./scripts/internal/doctor

All checks should succeed before continuing.

doctor only validates the host. It does not modify its configuration.

See Verify the host configuration.

6. Initialize the project

Run:

./scripts/init

init runs the prerequisite checks again, derives the project name and asks for an optional dotfiles repository.

It then creates the local .env configuration.

7. Create the devcontainer

Run:

./scripts/devcontainer up --recreate

Select:

hardened

for controlled sudo, selected SSH identities and optional dotfiles,

or:

bunkered

for the more restrictive environment without privilege escalation, SSH agent or dotfiles.

The first recreation also requests the GitHub token used by mise when required.

Hardened mode additionally requests:

* a sudo password for the recreation;
* the SSH identities to expose.

See Hardened vs Bunkered.

8. Connect

Once the environment is running:

./scripts/devcontainer ssh

The session starts in:

/workspaces/vscode

Daily commands

Start an existing environment:

./scripts/devcontainer up

Connect:

./scripts/devcontainer ssh

Reconfigure and recreate:

./scripts/devcontainer up --recreate

Delete the environment:

./scripts/devcontainer delete

See Daily workflow for the lifecycle semantics.

Next steps

Before using the environment with sensitive repositories, read:

* Security model
* SSH isolation
* Workspace isolation and denylist
* Linux privilege model
* Security considerations
