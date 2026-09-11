# Remote - GPU

The **Remote - GPU** extension is the client for the [MTGPU Server](https://mtcodeai.com/platform/gpu-server.html) — a server infrastructure management system that turns computers into clusters with automatic user account provisioning, comprehensive container support, failover job scheduler, and scalable architecture without ongoing administration. The extension provides a uniform view of all available servers with the live usage of each, one-click access to an SSH terminal, and integration with the Remote-SSH and Dev Containers extensions.

![Remote - GPU panel overview](images/screenshot-gpu-extension.png)

*The panel lists the available GPU servers and shows the selected server's details, live disk and GPU usage, and the **Remote-SSH** / **SSH Terminal** / **Container Workspace** / **Select Python** action buttons.*

## One Interface for Every GPU Server

Sign in once and the panel lists every GPU server you are authorized to use — even when they belong to different administrators.

- **User account provisioned automatically** — on first connection to any server, the MTGPU Server creates an unprivileged OS account for the user and sets up SSH key access automatically, no password required.
- **Select a server to work on** — see the server's identity, hardware, workspaces, and live usage. Access the server with a one-click SSH terminal, open a new window with the Remote-SSH extension, or attach to a workspace with the Microsoft Dev Containers extension.
- **Work on multiple servers simultaneously** — select another server to work on; all existing SSH terminals, Remote-SSH windows, and Dev Containers windows remain open.
- **Work from anywhere on the Internet** — both the MTGPU servers and the client are location agnostic, and each can stay in a private network behind a firewall without VPN or any network or firewall configuration. All communications travel over [DirectLink](https://mtcodeai.com/platform/platform.html) with P2P direct connection and end-to-end encryption.

## Getting Started

1. Create an [administrator account](https://mtcodeai.com/platform/get-started.html) and invite users to join.
2. Install the [MTGPU Server](https://mtcodeai.com/platform/gpu-server.html) on any computer and sign in with the administrator account.
3. Install this extension in VS Code and sign in with a user account.

## Requirements

- **Client:** VS Code 1.74+ on Windows, macOS, or Linux.
- **Host:** MTGPU Server on Linux or Windows WSL2.

## Learn More

- [MTGPU Server — full product overview](https://mtcodeai.com/platform/gpu-server.html)
- [How MTCode DirectLink works](https://mtcodeai.com/platform/platform.html)
- [MTCodeAI.com](https://mtcodeai.com)

---

© 2026 BrightTime Technologies, Inc. All rights reserved.
