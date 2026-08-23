FROM mcr.microsoft.com/devcontainers/base:bookworm

# Pre-create ~/.claude owned by the non-root user. devcontainer.json mounts a
# named volume here so Claude Code's auth token/settings survive rebuilds;
# without this, podman/docker initialize the volume as root-owned and the
# CLI silently fails to persist login state ("Failed to retrieve auth state
# after login").
RUN mkdir -p /home/vscode/.claude && chown -R vscode:vscode /home/vscode/.claude

# Add project-specific OS packages, language runtimes, or tools here.
# Example:
# RUN apt-get update && apt-get install -y --no-install-recommends \
#       <package-name> \
#     && rm -rf /var/lib/apt/lists/*
