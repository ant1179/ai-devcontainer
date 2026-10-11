# Quick Start

This guide provides the shortest path to creating and using a project
from the Secure AI Devcontainer template.

It assumes that all host prerequisites have already been installed and
configured.

If this is your first time using the template on this machine, complete
the host setup first:

[PREREQUISITES.md](PREREQUISITES.md)

For details about the architecture, security model and configuration,
see the [README](README.md).

------------------------------------------------------------------------

## 1. Create a project from the template

On GitHub, select:

``` text
Use this template
```

Create a new repository for your project.

Clone it:

``` bash
git clone <your-new-repository>
cd <your-project>
```

------------------------------------------------------------------------

## 2. Check the host

Run:

``` bash
./scripts/internal/doctor
```

All checks should succeed before continuing.

`doctor` validates that the host satisfies the requirements described in
[PREREQUISITES.md](PREREQUISITES.md). It does not modify the host
configuration.

------------------------------------------------------------------------

## 3. Initialize the project

Run:

``` bash
./scripts/init
```

`init`:

-   runs the prerequisite checks again;
-   derives the project name from the repository directory;
-   asks for an optional dotfiles repository;
-   creates the local `.env` configuration.

The `.env` file contains project configuration only. Secrets are stored
separately outside the workspace.

------------------------------------------------------------------------

## 4. Create the devcontainer

Run:

``` bash
./scripts/devcontainer up --recreate
```

You will be asked to select one of the two security modes.

### Hardened

Select `hardened` when you need a development environment with:

-   controlled `sudo`;
-   a dedicated SSH agent;
-   explicitly selected SSH identities;
-   optional personal dotfiles.

During recreation, you may be asked for:

-   the GitHub token used by `mise`;
-   a sudo password;
-   the SSH identities to expose.

### Bunkered

Select `bunkered` for the more restrictive environment:

-   no privilege escalation;
-   no SSH agent;
-   no personal dotfiles;
-   `no-new-privileges`;
-   an empty effective capability set for the development user.

The first recreation may request the GitHub token required by `mise`.

------------------------------------------------------------------------

## 5. Connect

Once the environment is running:

``` bash
./scripts/devcontainer ssh
```

The session starts in:

``` text
/workspaces/vscode
```

------------------------------------------------------------------------

## Daily commands

Start an existing environment without changing its configuration:

``` bash
./scripts/devcontainer up
```

Connect:

``` bash
./scripts/devcontainer ssh
```

Reconfigure and recreate:

``` bash
./scripts/devcontainer up --recreate
```

Delete the environment:

``` bash
./scripts/devcontainer delete
```

------------------------------------------------------------------------

## Command semantics

### `ssh`

``` bash
./scripts/devcontainer ssh
```

Connects to the existing workspace only.

It does not:

-   rebuild the image;
-   regenerate configuration;
-   recreate the workspace;
-   modify secrets.

### `up`

``` bash
./scripts/devcontainer up
```

Starts an existing workspace using its current configuration.

It does not rebuild or reconfigure the environment.

### `up --recreate`

``` bash
./scripts/devcontainer up --recreate
```

Runs the complete configuration path:

1.  select the security mode;
2.  prepare the required secrets;
3.  configure the dedicated SSH agent when using Hardened;
4.  build the image;
5.  generate the Compose overlay;
6.  recreate the DevPod workspace.

Use this command whenever the environment configuration needs to change.

### `delete`

``` bash
./scripts/devcontainer delete
```

Deletes the DevPod workspace and cleans up project-specific SSH agent
state.

Persistent project secrets stored outside the workspace are not
automatically removed.

------------------------------------------------------------------------

## Next steps

Before using the environment with sensitive repositories, read the
corresponding sections of the [README](README.md), especially:

-   Security model
-   Hardened vs Bunkered
-   Secrets
-   SSH isolation
-   Workspace isolation and denylist
-   Linux privilege model
-   Security considerations
