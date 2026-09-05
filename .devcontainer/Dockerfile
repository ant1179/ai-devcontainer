FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

COPY --from=jdxcode/mise /usr/local/bin/mise /usr/local/bin/

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        acl \
        sudo \
    && rm -rf /var/lib/apt/lists/*

# Remove passwordless sudo potentially provided by the base image.
RUN rm -f /etc/sudoers.d/vscode \
    && usermod -aG sudo vscode

# Set vscode password from a pre-computed hash.
RUN --mount=type=secret,id=vscode_password_hash \
    test -s /run/secrets/vscode_password_hash \
    && usermod --password "$(cat /run/secrets/vscode_password_hash)" vscode

# Always require the sudo password and do not cache authentication.
RUN printf '%s\n' \
    "Defaults:vscode timestamp_timeout=0" \
    "Defaults:vscode use_pty" \
    > /etc/sudoers.d/99-vscode-hardened \
    && chmod 0440 /etc/sudoers.d/99-vscode-hardened

# Install tools managed by mise using a temporary GitHub token.
COPY --chown=vscode:vscode mise.toml /tmp/mise.toml

USER vscode

RUN --mount=type=secret,id=mise_github_token \
    export MISE_GITHUB_TOKEN="$(cat /run/secrets/mise_github_token)" \
    && cd /tmp \
    && mise trust mise.toml \
    && mise install \
    && rm -f /tmp/mise.toml

# Make sure mise is activated in both zsh and bash.
RUN echo 'eval "$(mise activate bash)"' >> /home/vscode/.bashrc \
    && echo 'eval "$(mise activate zsh)"' >> /home/vscode/.zshrc

USER vscode
