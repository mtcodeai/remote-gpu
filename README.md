# Remote - GPU

> **Preview release.** This extension connects to computers running [MTGPU Server](https://mtcodeai.com/platform/gpu-server.html). The MTGPU Server download is coming soon on mtcodeai.com; until then the extension is usable only with servers an administrator has already set up.

The **Remote - GPU** extension is the client for [MTGPU Server](https://mtcodeai.com/platform/gpu-server.html), which turns existing workstations or cloud hosts into a multi-tenant cluster with a simple configuration. Installed in minutes, the cluster forms automatically, and user accounts are provisioned on first access without administrator setup. Existing OS accounts keep working without interruption. The built-in job scheduler handles native and containerized jobs and fails over on its own—no separate scheduling system to install and maintain. It scales from a single server to a cluster of thousands of servers with the same setup and user experience: add computers or shared storage as the need arises, and user file ownership travels from server to server with no account or data migration. The extension provides a uniform view of the available servers with the live usage of each, one-click access to an SSH terminal, and integration with the Remote-SSH and Dev Containers extensions.

![Remote - GPU panel overview](images/screenshot-gpu-extension.png)

*The panel lists the available GPU servers and shows the selected server's details, live disk and GPU usage, and the **Remote-SSH** / **SSH Terminal** / **Container Workspace** / **Select Python** action buttons.*

## One Interface for Every GPU Server

Sign in once and the panel lists every GPU server you are authorized to use — even when they belong to different administrators.

- **User account provisioned automatically** — on first connection to any server, the MTGPU Server creates an unprivileged OS account for the user and sets up SSH key access automatically, no password required.
- **Consistent user identities** — MTGPU provisions tenant accounts with unique UID/GIDs that are consistent across all servers, allowing seamless home-directory and storage sharing.
- **Select a server to work on** — see the server's identity, hardware, workspaces, and live usage. Access the server with a one-click SSH terminal, open a new window with the Remote-SSH extension, or attach to a workspace with the Microsoft Dev Containers extension.
- **Work on multiple servers simultaneously** — select another server to work on; all existing SSH terminals, Remote-SSH windows, and Dev Containers windows remain open.
- **Submit jobs and manage workspaces** — in an SSH terminal, use `mt-job` to submit jobs to the server or to the cluster scheduler, including multi-node jobs for distributed training and array jobs for parameter sweeps; use `mt-container` to create, start, stop, or remove container workspaces.
- **Work from anywhere on the Internet** — servers and clients stay in their own private networks behind firewalls. No VPN setup, no port forwarding, no firewall opening. All communications travel over [DirectLink](https://mtcodeai.com/platform/platform.html) with P2P direct connection and end-to-end encryption.

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
