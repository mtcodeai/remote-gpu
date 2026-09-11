# mtgpu-server

Multi-tenant GPU server on the MTCode DirectLink platform. Makes a GPU machine
behind NAT reachable from anywhere and gives each tenant an isolated OS account
reachable over SSH (VS Code Remote-SSH, Dev Containers, etc.).

This is the new, simplified design described in
`remoteGPU-cpp-server/New design of GPU server`. It replaces the Python
project/execution machinery of the old `remoteGPU-cpp-server` with per-user OS
accounts + a dedicated OpenSSH server.

## What it does

At startup (`src/main.cpp`):

1. Loads `config/config.json`, creates `ROOT_FOLDER` (+ `Private/`, `.mtgpu/`).
2. Loads the GPU-server account credentials (secure store slot
   `MTCODE_USERNAME`/`MTCODE_PASSWORD` env vars, or the `MTCODE_GPU_SERVER`
   secure-store slot when run as the user that saved them).
3. Registers the GPU control server with DirectLink (`REGISTRATION_GPU_SERVER`,
   appId `GPU-Server`).
4. Starts a dedicated loopback-only `sshd` on the first free port at or above
   2222, with a generated config and host keys (`PasswordAuthentication no`,
   key-only). The port is not a setting; the DirectLink registration in the
   next step carries it to clients.
5. Launches `mtserver-cli` to register that sshd with DirectLink
   (`MTSERVER_APPID=GPU_SERVER_SSH`, `--protocol SSH --start`).
6. Accepts tenant connections; on first contact for an account it creates the
   OS account + home directory on demand and applies access rules.
7. A background job removes files **and** the OS account after
   `TENANT_RETENTION_DAYS` of inactivity (unless a `REQUEST_FILE_KEEP` window is
   active).

## Per-tenant provisioning

For account `demo`: home = `ROOT_FOLDER/Private/demo`, an OS account `demo` is
created with a **locked password** (SSH-key auth only). Dataset files get
read-only access with only directory traversal/list permissions where needed,
configured Python interpreters get read+execute, and a per-user disk
quota is applied when supported. Public keys arrive via `REGISTER_SSH_KEY` and
are appended to `~/.ssh/authorized_keys` (de-duplicated).

### Default account and Python environment

Provisioning runs when an authenticated tenant first connects and is
idempotently refreshed on later connections. With the example configuration,
account `demo` receives the following defaults:

| Setting | Result |
|---------|--------|
| Home directory | `/home/zezhen/GPU_SERVER_DISK/Private/demo` (`ROOT_FOLDER/Private/<account>`) |
| Home/runtime access | Owned by `demo`; mode `700` on Linux/macOS. Configured `ADMIN_USERS` receive read/traverse ACL access to tenant homes and MTGPU-managed runtime stores. |
| Login | The dedicated SSH server accepts public keys only. Linux/macOS lock the OS password; Windows uses an undisclosed generated password required for account creation. Administrators can use platform-native account tools to set a local-console password when needed. |
| SSH keys | Stored in `~/.ssh/authorized_keys`; `.ssh` is mode `700` and the file is mode `600` on Linux/macOS. |
| Shared data | `DATASETS_DIRECTORIES` files are read-only without file execute permission; directories receive the traversal/list permissions needed to reach files. On Linux/macOS they also appear by basename under `~/shared`. |
| Per-user grants | `GRANT_USER_ACCESS` can grant named tenants additional host paths with `R`, `W`, and/or `E` rights. These grants are applied during provisioning so intentional exceptions survive server restarts and account refreshes. |
| Exposed commands | `EXPOSE_COMMANDS` maps command names to absolute host executable paths. Valid commands are available by name in MTGPU tenant SSH shells and native jobs. |
| Python interpreters | `PYTHON_INTERPRETERS` are read-execute. On Linux/macOS they also appear by basename under `~/shared`. |
| Private Python | Each host keeps the tenant's environments, catalog, and selection under `<SERVER_RUNTIME_DIR>/python-venvs/<tenant>`. The shared `~/.venv` link resolves through that host's local `active` link, and `~/.local/bin/python` follows the same selection. |
| Storage | `USER_DISK_LIMIT` is the default per-user soft quota; `USER_DISK_LIMITS` can override specific accounts; `USER_DISK_SOFT_EXTRA` adds grace headroom before the hard stop. Quotas are applied at provisioning and re-applied for known tenants at server startup where the host filesystem supports it. |
| Retention | The account and its home, managed Python venv store, Remote-SSH server store, and container workspace state are removed after `TENANT_RETENTION_DAYS` of inactivity unless a keep window is active. |

`PYTHON_INTERPRETERS` contains structured definitions for the host Python
environments offered to tenants. Each entry has a stable `id`, a user-facing
`name`, and its absolute `path`. `DEFAULT_PYTHON_INTERPRETER` contains the ID
selected for new accounts. `path` may be an environment directory or the
Python executable itself. A directory resolves to `bin/python` on Linux/macOS
and to `Scripts/python.exe` (or `python.exe` at the directory root) on Windows.

For example:

```json
"PYTHON_INTERPRETERS": [
  {
    "id": "pytorch-cu126",
    "name": "Python 3.12 - CUDA 12.6 (PyTorch 2.12)",
    "path": "/home/zezhen/venvs/pytorch-cu126"
  },
  {
    "id": "python-system-3-12",
    "name": "Python 3.12 - System (CPU)",
    "path": "/usr/bin/python3.12"
  }
],
"DEFAULT_PYTHON_INTERPRETER": "pytorch-cu126"
```

Keep `id` stable because it is used in protocol requests and tenant directory
names. Use `name` to identify the Python version and, for GPU environments, the
CUDA and framework versions. See `multi-interpreters-design.md` for the full
implementation contract.

`EXPOSE_COMMANDS` exposes selected host executables without adding their entire
parent directories to tenant `PATH`. For example, WSL2 normally provides
`nvidia-smi` outside the default Linux path:

```json
"EXPOSE_COMMANDS": {
  "nvidia-smi": "/usr/lib/wsl/lib/nvidia-smi"
}
```

The server creates host-namespaced managed links below
`~/.mtgpu/exposed-commands/<server-id>` and adds that directory to MTGPU SSH
sessions and native jobs. The installer and server warn about missing,
non-regular, or non-executable paths; invalid entries are skipped. This setting
does not change filesystem permissions on the target.

SSH-key registration refreshes these links and the managed shell block before
returning success, including when `ACCOUNT_NAME_MAP` maps an MTCode identity to
an existing OS account. The first terminal opened after registration therefore
uses the current member's command namespace rather than waiting for deferred
full provisioning.

For each entry, the server creates a tenant-owned venv using
`venv --system-site-packages`. On Linux its authoritative path is
`<SERVER_RUNTIME_DIR>/python-venvs/<tenant>/<interpreter-id>`. The per-tenant
directory is owned by the tenant and mode `700`. Catalog and selection state
are kept beside those environments so two GPU hosts never overwrite each
other through a shared tenant home.

It also writes `mtgpu-default-python.pth` in that environment's site-packages
containing the corresponding base interpreter's site-package directories. The
resulting package model is:

- Packages already installed in the configured interpreter, such as PyTorch,
  are visible to every tenant but remain in the shared, administrator-managed
  installation.
- Each managed environment and its site-packages are owned by the tenant. After activation,
  ordinary `pip install <package>` installs additional packages there without
  writing to the shared interpreter or the OS Python installation.
- Packages installed by one tenant are not visible to another tenant. Removing
  the tenant account also removes its installed packages.

The active interpreter is recorded in the host-local
`<SERVER_RUNTIME_DIR>/python-venvs/<tenant>/default-python`; the adjacent `active`
link points to the selected environment. The shared-home link has a stable
target:

```text
~/.venv -> <SERVER_RUNTIME_DIR>/python-venvs/<tenant>/active
~/.local/bin/python -> <SERVER_RUNTIME_DIR>/python-venvs/<tenant>/active/bin/python
```

`SERVER_RUNTIME_DIR` must have the same absolute value on every Linux/WSL
cluster member while resolving to local storage on each computer. Selections
and installed packages are therefore per-host even when `HOME` is shared.

On Linux, the server appends a managed block to both `~/.profile` and
`~/.bashrc`; on macOS it updates those files plus `~/.zprofile` and `~/.zshrc`.
The block sets:

```sh
export VIRTUAL_ENV="$HOME/.venv"
export PATH="$VIRTUAL_ENV/bin:$PATH"
```

Consequently, a normal interactive SSH shell resolves `python` through the
host's selected environment. `python -m pip` is the recommended package
command because it is unambiguous even when a distribution does not install
an unversioned `pip` link. Prefer the interpreter-explicit form when
diagnosing an environment mismatch:

```sh
python -m pip install opencv-python
python -c "import sys, cv2; print(sys.executable); print(cv2.__file__)"
```

Windows provisioning creates `~/activate_mtgpu_python.cmd` and
`~/Activate-MTGpuPython.ps1`; it does not modify a global user `PATH`. Run the
appropriate script in each new Command Prompt or PowerShell session before
using `python` or `pip`.

The MTGPU extension's **Select Python** action queries the allowlisted
interpreters and current selection from the selected server. Choosing another
entry updates that host's local `active` link; clients never submit arbitrary
host paths. Cluster UIs should group the responses by server. The scheduler
also receives each member's ready interpreter IDs and restricts a native
`--python <id>` job to members advertising that ID.
Restart existing terminals, Python language servers, and debug sessions after
switching because already-running processes retain their original environment.

The server also provisions a standalone C++ command into each account, so an
SSH terminal does not require the MTGPU extension:

```sh
mt-python list
mt-python current
mt-python use pytorch-cu126
```

The command reads the current host's
`<SERVER_RUNTIME_DIR>/python-venvs/<tenant>/python-interpreters.json` and accepts only
listed IDs. `mt-python list` identifies the server whose catalog it is showing.
The executable itself is installed in the host-local store; the shared
`~/.local/bin/mt-python` path is only a stable link to that local copy.
Rebuild/restart the server and reconnect an existing account once to migrate
the catalog, selection, command, `~/.venv`, and `python` links.

#### Python environment caveats

- Every parent directory needed to reach each `PYTHON_INTERPRETERS` entry must
  be traversable by tenant accounts. The server applies ACLs to the configured
  directory, but cannot repair inaccessible parent directories.
- Provisioning does not recreate a managed environment when its `pyvenv.cfg`
  exists. Moving a shared environment or upgrading it to a different Python
  minor version may invalidate existing tenant environments. Back up project
  requirements, remove the affected host-local
  `<SERVER_RUNTIME_DIR>/python-venvs/<tenant>/<id>`, reconnect to
  recreate it, and reinstall tenant packages.
- Shared packages can constrain or conflict with tenant-installed dependency
  versions. Use `python -m pip`, `python -m pip show <package>`, and inspect
  `package.__file__` to confirm which copy is active. Do not run pip against the
  shared interpreter path unless intentionally administering the shared image.
- Native wheels must match the base interpreter, operating system, CPU, and,
  where applicable, the installed CUDA/driver stack. Visibility through the
  `.pth` file does not make an ABI-incompatible package usable.
- Shell startup files only affect shells that read them. VS Code, debug
  configurations, services, cron jobs, and non-interactive commands may select
  another interpreter. Configure those tools explicitly with
  `~/.venv/bin/python` (or `~/.venv/Scripts/python.exe` on Windows).
- `pip` installs third-party distributions only. Standard-library modules such
  as `runpy` already ship with Python and cannot be installed separately.
- Virtual-environment creation requires a working `venv` module. Debian/Ubuntu
  hosts may need the matching `python3-venv` package installed by the server
  administrator.
- Linux quota enforcement requires quota-enabled storage and tools such as
  `setquota`; macOS and Windows behavior depends on host volume configuration.
  A configured limit is therefore not proof that the filesystem enforces it.
  On startup, the server re-applies `USER_DISK_LIMIT`,
  `USER_DISK_SOFT_EXTRA`, and `USER_DISK_LIMITS` to known tenants, so admins
  can change quota policy and restart the server.
- On Linux deployments, tenant Python venvs are stored under
  `<SERVER_RUNTIME_DIR>/python-venvs/<tenant>` rather than physically inside
  `ROOT_FOLDER`. The panel's measured usage includes that store, and cleanup
  removes it, but kernel hard quotas on `ROOT_FOLDER` do not automatically
  enforce limits on `/var/lib/mtgpu`.

### Linux managed runtime storage

On Linux and WSL2, tenant project files remain under
`ROOT_FOLDER/Private/<tenant>`, while runtime data that needs predictable Linux
filesystem semantics is kept under server-provisioned paths:

| Data | Location | Reason |
|------|----------|--------|
| Tenant project files | `ROOT_FOLDER/Private/<tenant>` | Administrator-chosen project storage; can be a large Windows-backed drive on WSL2. |
| Local control socket | `/run/mtgpu/<server>-<root-hash>/control.sock` | Unix sockets are unreliable on DrvFS. |
| Rootless container storage | `/var/lib/mtgpu/containers/<tenant>` | Overlay/chown/container metadata require Linux semantics. |
| Python venvs | `<SERVER_RUNTIME_DIR>/python-venvs/<tenant>` | Compiled wheels and pip metadata need Linux filesystem behavior. |
| Remote-SSH server installs | `<SERVER_RUNTIME_DIR>/vscode-server/<tenant>` | Provides a tenant-owned, server-provisioned install root for editor servers. |
| Dev Containers helper installs | `<SERVER_RUNTIME_DIR>/vscode-remote-containers/<tenant>` | Keeps Dev Containers setup files out of project storage and on Linux-native storage. |

The managed runtime paths are tenant-owned where tenants write to them, and per-tenant
directories are mode `700` when they contain private package/editor state. The
GPU panel includes managed Python venv bytes in tenant disk usage reporting.

On WSL2, default DrvFS automounts expose Windows drives broadly under
`/mnt/c`, `/mnt/d`, and similar paths. `install_server.sh` now offers a WSL
drive-isolation step that writes `/etc/wsl.conf` automount options:
`metadata,uid=0,gid=<mtgpu-admins-gid>,umask=027,fmask=027,dmask=027` (file
execute stays enabled for root/admins so WSL-to-Windows interop keeps working;
tenants get no access, which also blocks tenant interop). After the required WSL
restart, Windows-drive mounts are root-owned/private by default. Linux account
provisioning grants tenant traverse ACLs only along `/mnt/<drive>` parent chains
needed for `ROOT_FOLDER`, `DATASETS_DIRECTORIES`, and any configured
`GRANT_USER_ACCESS` paths, then grants the requested access inside the allowed
home/dataset/grant trees. This is the WSL2 mechanism that turns configured
paths into an allowlist instead of merely an extra grant. The
installer treats the Linux `acl` package (`setfacl`/`getfacl`) as a host
prerequisite because those ACL grants are required after `/mnt/<drive>` is made
root-private. Because DrvFS metadata support can vary across Windows-drive
mount modes, provisioning also applies a WSL-only fallback that does not depend
on ACLs: approved parent directories are owned by the `mtgpu-admins` group with
`rwxr-x--x` so administrators keep read+traverse while tenants traverse through
via the "other" execute bit only (no read/list), and configured dataset trees
receive read/list access via the `mtgpu-tenants` group. Per-user
`GRANT_USER_ACCESS` entries intentionally do not use that shared-group fallback;
if ACLs are unavailable, the server logs the failed grant instead of exposing
the path to every tenant. macOS applies these grants with filesystem ACLs, and
Windows applies them with `icacls`.

**Warning — cross-tenant exposure when isolation is skipped.** If isolation is
skipped (`--skip-wsl-drive-isolation` or declining the prompt) and `ROOT_FOLDER`
or `DATASETS_DIRECTORIES` are on a Windows drive, that mount has no `metadata`
option, so the per-tenant home `chmod 700` cannot persist. Tenant homes stay
`777` and **tenants can read each other's home and project files.** Do not skip
isolation on a shared host with tenant data on `/mnt/<drive>`; keep `ROOT_FOLDER`
on Linux-native storage or enable isolation. `install_server.sh` detects whether
`ROOT_FOLDER` is on a Windows drive and escalates its skip warning to CRITICAL in
that case. On Linux-native storage `chmod 700` works and tenants stay isolated
regardless of `/mnt/<drive>` isolation.

Remaining design caveat: hard filesystem quotas are still mount-specific. If
`ROOT_FOLDER` is on `/mnt/d`, a kernel quota on that filesystem does not
automatically limit `<SERVER_RUNTIME_DIR>/python-venvs` or
`/var/lib/mtgpu/containers`. Use measured usage reporting for visibility, and
configure separate filesystem/project quotas on Linux-native stores if hard
enforcement is required there.

### Disk quota enablement and enforcement

Disk quota support has two separate stages:

1. **Host filesystem capability** is enabled once by the administrator:

   ```bash
   ./install_server.sh --skip-container
   ```

   The installer reads `ROOT_FOLDER`, finds the filesystem that contains it,
   installs quota tools such as `setquota`, updates `/etc/fstab` for ext4 with
   `usrquota,grpquota`, remounts the filesystem, runs `quotacheck`, and enables
   quota accounting with `quotaon`. This makes the OS capable of hard quota
   enforcement across reboots. It does **not** apply tenant quota policy.

2. **Tenant quota policy** is applied by `mtgpu-server` at runtime. On account
   provisioning and on every elevated server startup, the server reads:

   ```json
   "USER_DISK_LIMIT": 100,
   "USER_DISK_SOFT_EXTRA": 10,
   "USER_DISK_LIMITS": {
     "demo": 200,
     "student-a": 50
   }
   ```

   `USER_DISK_LIMIT` is the default soft quota in GB. `USER_DISK_LIMITS`
   overrides specific tenant accounts; an override of `0` disables the quota
   for that account. `USER_DISK_SOFT_EXTRA` adds a grace band above the soft
   quota before the hard limit. With the example above, most accounts get a
   soft quota of 100 GB and a hard limit of 110 GB; `demo` gets a soft quota
   of 200 GB and a hard limit of 210 GB. The server calls `setquota` on the
   filesystem containing each tenant home. This means administrators can change
   quota values in `config.json` and restart `mtgpu-server` to reapply them.

To verify hard enforcement on Linux:

```bash
findmnt -T /path/to/ROOT_FOLDER -o TARGET,SOURCE,FSTYPE,OPTIONS
sudo repquota <mountpoint>
sudo quota -u <tenant-account>
```

In `quota -u`, the `quota` column is the soft quota and the `limit` column is
the hard stop. If soft quota is exceeded but hard quota is not, the filesystem
shows a grace period. Once the hard limit is reached, writes fail immediately.

If the panel shows usage above the configured limit, existing files are not
removed automatically. With hard quotas active, further writes or file growth
should fail until the tenant reduces usage below the quota. Without hard quotas
active, the panel can warn but the OS may still allow writes.

## Messages handled

`REGISTER_SSH_KEY` (new), `MSG_TYPE_GET_GPU_INFO`,
`MSG_TYPE_QUERY_EXECUTION_STATUS` (now returns the **account names** of running
GPU processes), `MSG_TYPE_QUERY_PYTHON_INTERPRETERS`,
`MSG_TYPE_SET_DEFAULT_PYTHON_INTERPRETER`, `MSG_TYPE_REQUEST_FILE_KEEP`, and
`MSG_TYPE_QUERY_CONTAINER_CAPABILITIES`, the container workspace query/create/
start/stop/remove messages, and `MSG_TYPE_HEARTBEAT` (now answered).

## Build

```sh
./build_server.sh
```

This configures the repository-root `CMakeLists.txt` in
`build/server-release` and builds `mtgpu-server`, `mt-python`, and
`mt-container`. Set `BUILD_DIR`, `BUILD_TYPE`, or `JOBS` to override the
defaults.

Reuses utility code from `../remoteGPU-cpp-server` (GPU/disk usage,
`NetworkUtils`) and the DirectLink stack from `../shared`. Requires OpenSSL and
libcurl. Override the source roots with `-DMTCODE_ROOT=...`.

## Smoke Tests

```sh
./scripts/test_cluster_shared_home.sh
```

This exercises the automatically testable shared-home/admin CLI behavior:
`mtgpu-admin` help text, config path handling, `TENANT_HOME_ROOT`,
`SERVER_RUNTIME_DIR`, disk-usage reporting, and invalid
tenant-name rejection. Run it with `sudo` to include root-only checks for
cluster-mode shared-home deletion guards, host-local runtime cleanup, and
tenant-removal audit records.

## Run

```sh
sudo ./build/server-release/mtgpu-server --config config/config.json
```

Must run **elevated** (root / Administrator) to manage OS accounts, quotas and
`.ssh` files. Without elevation it warns and account operations fail.

## Design decisions vs. the original draft

- **SSH port 2222** (first free port from 2222 upward), not 25 (25 is the
  privileged SMTP port). The port is chosen at startup, not configured.
- **SSH-key-only auth**: Linux/macOS accounts are created with a locked
  password. MTGPU does not generate or retain recoverable tenant passwords;
  administrators can use platform-native account tools to set one when local
  console access is required.
- **Cleanup removes files + OS account** after `TENANT_RETENTION_DAYS`.
- **Admin maintenance CLI**: `mtgpu-admin list-users`, `disk-usage [tenant]`,
  and `remove-user <tenant>` provide explicit tenant inspection and destructive
  removal. `disk-usage` prints `?` for runtime stores the invoking OS user
  cannot read, or appends `*` when only a partial size could be counted.
  `remove-user` also clears hybrid Python, Remote-SSH, Dev Containers, and
  container stores plus stale workspace/reservation state.
- **All three platforms** (Linux/Windows/macOS) via `UserAccountManager`
  implementations. Linux is fully exercised; Windows/macOS use `net user` /
  `icacls` / `fsutil` and `dscl` respectively and need on-target validation.

## Container user manual

Container workspaces are optional. A user can continue working directly in the
native Remote-SSH environment, or create one managed container workspace for a
reproducible Python/CUDA environment. Both modes use the same account, private
home, projects, shared datasets, SSH endpoint, and GPU server.

The server administrator must first enable `CONTAINERS` in `config.json`, build
the server with container support, and prepare the host with
`./install_container.sh`. When the selected server reports container support as
ready, users can manage the same workspace from either the MTGPU extension or a
regular SSH shell.

### Using containers from the MTGPU extension

1. Sign in and select a GPU server from the **Available GPU Servers** table.
2. Confirm that **Containers** reports an available rootless Podman or Docker
   runtime. If it reports unavailable, the displayed reason must be resolved by
   the server administrator.
3. Select **Container Workspace**. For a new workspace, choose one of the
   administrator-selected templates. The first creation may take several minutes while
   its OCI image is downloaded into the tenant's rootless image store.
4. Select **Container Workspace** again after creation and choose an action:
   - **Open Terminal** opens an interactive shell inside the running container.
   - **Show Logs** opens the most recent output from the container's main
     process.
   - **Manage Processes** lists in-container processes and can send `TERM`,
     `INT`, `HUP`, or `KILL` to a selected PID.
   - **Show Volumes** lists persistent named volumes and their container mount
     paths when the administrator has configured any.
   - **Attach with Dev Containers** hands the running container to the Dev
     Containers extension. From a local window, MTGPU opens Remote-SSH and then
     automatically attaches the selected workspace by its full runtime ID;
     no second panel action or generic container picker is required.
     This action is enabled only in official Microsoft VS Code because
     `ms-vscode-remote.remote-containers` is not distributed through Open VSX.
     MTCode Studio displays a non-selectable
     **Attach with Dev Containers (not official VS Code)** entry instead.
   - **Stop** shuts down the container without deleting it.
   - **Start** restarts an existing stopped container.
   - **Remove** deletes the container and its writable image layer after
     confirmation. Files in the private home and shared datasets are preserved.

Up to `CONTAINERS.maxWorkspacesPerTenant` managed workspaces may exist for each
tenant. Their state, queue position, GPU assignment, and template appear in the
panel regardless of whether they were created from the extension or from
`mt-container` in SSH.

### Using containers from an SSH shell

After the server is rebuilt and restarted, reconnect once so account
provisioning installs `mt-container` and refreshes the allowed-template
policy. List the configured templates, tenant-local saved images, and volumes:

```bash
mt-container templates
mt-container volumes
```

Create and enter a persistent workspace:

```bash
mt-container create pytorch-2.4-cuda12.4
mt-container list
WS_ID=<id-from-list>
mt-container status "$WS_ID"
mt-container ps "$WS_ID"
mt-container logs "$WS_ID"
mt-container shell "$WS_ID"
```

Run a single command without opening an interactive container shell:

```bash
mt-container exec "$WS_ID" -- python /workspace/Projects/example/train.py
```

For `exec`, use container paths such as `/workspace/...`; the outer SSH shell
would expand `~` to the host home before the command reaches the container.

Manage its lifecycle from the SSH shell:

```bash
mt-container stop "$WS_ID"
mt-container start "$WS_ID"
mt-container kill "$WS_ID" 1234 TERM
mt-container remove "$WS_ID"
```

Templates are a preselected convenience catalog, not an image allowlist. When
the administrator enables `allowCustomImages`, the extension,
`remotegpu-cli`, and `mt-container` let users pull any accessible public or
already-authenticated OCI image:

```bash
mt-container create --image docker.io/nvidia/cuda:12.4.1-base-ubuntu22.04
```

The GPU Panel exposes the same workflow as **Use Custom Image...**. Custom
images receive the same rootless runtime, GPU, network, home,
dataset, and persistent-volume policy as curated templates. Registry login and
credential distribution remain administrator/user responsibilities.

Tenant-owned `devcontainer.json` files can select an image, build a Dockerfile,
or define a constrained Docker Compose primary service, and publish VS Code
customizations:

```jsonc
{
  "name": "Project environment",
  "image": "docker.io/library/ubuntu:24.04",
  "workspaceFolder": "/workspace",
  "customizations": {
    "vscode": {
      "settings": {
        "terminal.integrated.defaultProfile.linux": "bash"
      }
    }
  }
}
```

Create it from SSH with
`mt-container create --devcontainer .devcontainer/devcontainer.json`, or
select **Use devcontainer.json...** in the GPU Panel and enter the path relative
to the GPU-server home. The file must remain inside that home. This release
accepts comments but not trailing commas, and supports only `$schema`, `name`,
`image`, `build`, `dockerComposeFile`, `service`, `workspaceFolder`, and
`customizations`; `workspaceFolder`, when present, must be `/workspace`. A
`build` string names a Dockerfile, while a build object accepts `dockerfile`,
`context`, and string-valued `args`. Build paths and Compose files must stay
inside the tenant home. Compose support is limited to one selected primary
service; MTGPU injects its policy through an override file. Other properties
fail explicitly rather than being silently ignored.

Each successful `create` allocates a new workspace ID until the per-tenant cap
is reached. Lifecycle commands accept that ID; they may omit it only when
exactly one workspace exists. `remove` never deletes the tenant's home or
shared datasets.

### Paths, packages, and networking

- The private home is mounted read-write at `/workspace`, with
  `HOME=/workspace`. Consequently, `~/Projects`, `~/shared`, and other
  home-relative paths work naturally inside the container.
- Configured dataset directories are mounted at their original absolute paths
  and remain read-only. Existing links such as `~/shared/Datasets` therefore
  resolve the same way in native SSH and in the container.
- Matching tenant package directories are added to `PYTHONPATH` for templates
  that publish a Python ABI tag such as `py312`. Packages installed normally
  inside the container remain across stop/start but are part of its writable
  layer and disappear when the workspace is removed.
  For example, in an Ubuntu-based workspace:

  ```bash
  apt update
  apt install -y curl
  curl -I https://mtcodeai.com
  ```

  `curl` remains after `mt-container stop <id>` /
  `mt-container start <id>`, but it is gone after
  `mt-container remove <id>` and recreate because removal deletes the
  container's writable layer. Put always-needed OS packages in a Dockerfile,
  devcontainer build, or custom image instead:

  ```Dockerfile
  FROM docker.io/library/ubuntu:24.04
  RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
  ```
- With `networkPolicy` set to `disabled`, package downloads such as
  `pip install` cannot reach the Internet. The administrator must use an image
  containing the dependency, provide it through the mounted tenant packages,
  or explicitly enable the `egress` policy and recreate the workspace.
- Current managed workspaces expose reserved GPUs through vendor CDI selectors
  when a matching CDI spec exists. Verify access with a vendor runtime check
  inside the container, for example `nvidia-smi`, `rocm-smi`, `sycl-ls`, or the
  framework's CUDA/ROCm/oneAPI availability check.
- CPU, RAM, and GPU requirements belong to each job rather than the container
  policy. Managed containers therefore use the engine's CPU/RAM defaults.
  `pidsLimit` remains a server safety guard against unbounded task creation.
- Administrator-configured named volumes are mounted read-write at their
  declared container paths. They survive **Remove** and template changes, but
  are deleted when the tenant account expires or is administratively cleaned
  up. Use `mt-container volumes` to list them.

Users may run raw `podman` or `docker` commands when their account permits it,
but independently named containers are outside MTGPU management: they do not
appear in the panel, do not receive the server's mounts and policy
automatically, and are not removed by the managed-workspace lifecycle.

## Container feature test guide

This section is an acceptance-test checklist for administrators and users. Run
administrator commands on the GPU host from this repository. Run
`mt-container` commands as the provisioned tenant in an SSH terminal, not as
root and not from a shell inside the container. The CLI is a host-side broker
for the tenant's rootless Podman/Docker runtime and is intentionally not
installed or mounted inside managed containers.

Examples below use tenant `demo`, Podman, and `/mtgpu/cache`. Substitute the
actual account, engine, template IDs, dataset paths, and volume paths. With
Docker, replace `podman` with `docker` in inspection commands. `current` is not
a container CLI command; use `mt-container status`.

After creating a workspace, set these shell variables for inspection examples:

```bash
mt-container list
WS_ID=<id-from-list>
WS_NAME=$(mt-container status "$WS_ID" | jq -r .name)
```

For example, if `create` prints `Workspace 1 is running (...)` or `list` shows
an `ID` row of `1`, use `WS_ID=1`. `WS_ID` is the broker-facing workspace ID.
`WS_NAME` is the generated, globally unique Podman/Docker container name. Do not
assume a fixed runtime name.

### 1. Prepare the host and test configuration

1. Build all container components:

   ```bash
   cd /home/zezhen/mtcodeai/mtgpu-server
   ./build_server.sh
   ```

2. Check or install the Linux host prerequisites:

   ```bash
   ./install_container.sh
   ```

   The final preflight must find a rootless Podman or Docker engine. On NVIDIA
   hosts it also checks NVIDIA Container Toolkit/CDI and runs a GPU smoke test.
   On non-NVIDIA or CPU-only hosts the installer skips NVIDIA setup and leaves
   vendor GPU CDI generation to the administrator. Review every requested
   privileged change before approving it.

3. Enable a representative test policy in `config/config.json`. Start with a
   public image and a disposable volume; do not put registry passwords or other
   secrets in this file:

   ```json
   "CONTAINERS": {
     "enabled": true,
     "allowCustomImages": true,
     "networkPolicy": "disabled",
     "pidsLimit": 256,
     "maxWorkspacesPerTenant": 4,
     "volumes": [
       {
         "name": "manual-test-cache",
         "mountPath": "/mtgpu/cache"
       }
     ],
     "templates": []
   }
   ```

4. Restart the elevated server with the same configuration used in production:

   ```bash
   sudo ./build/server-release/mtgpu-server --config config/config.json
   ```

5. Reconnect the `demo` account from the MTGPU extension. Provisioning is
   idempotent and refreshes `~/.local/bin/mt-container` and
   `~/.mtgpu/container-policy.json`.

6. In the tenant's SSH terminal, verify the refreshed policy:

   ```bash
   command -v mt-container
   jq '{enabled,allowCustomImages,networkPolicy,pidsLimit,detectedGpuCount,
        maxWorkspacesPerTenant,controlSocket,volumes}' \
     ~/.mtgpu/container-policy.json
   ```

   Expected: the command resolves from the tenant account and the printed
   values match `config.json`. If `jq` is unavailable, use
   `sed -n '1,200p' ~/.mtgpu/container-policy.json`.

7. Begin from a clean managed workspace:

   ```bash
   for id in $(mt-container list | awk 'NR > 1 && $1 ~ /^[0-9]+$/ {print $1}'); do
     mt-container remove "$id"
   done
   ```

### 2. Optional backend and capability detection

1. In the SSH terminal, list the available workspace images and query the
   absent workspace:

   ```bash
   mt-container templates
   mt-container status
   ```

   Expected: configured templates and any usable tenant-local images are
   printed; status reports that no workspaces exist. Engine/version/rootless
   capability appears in the GPU Panel and the server's container capability
   response.

2. In the MTGPU extension, select the server. The **Containers** row must show
   the same engine rather than `Unavailable`.

3. Negative test: set `CONTAINERS.enabled` to `false`, restart, and reconnect.
   The panel must hide/disable container management while native SSH,
   `mt-python list`, and Python selection continue to work. Restore
   `enabled: true`, restart, and reconnect before continuing.

### 3. Curated templates, image pulling, and workspace IDs

1. Pick an ID printed by `templates` and create it:

   ```bash
   mt-container create <template-id>
   mt-container list
   ```

   Expected: the first run pulls the image if necessary, creates a uniquely
   named workspace, starts it, and reports its ID and selected template. Set
   `WS_ID` and `WS_NAME` as shown above.

2. Run the same create command again:

   ```bash
   mt-container create <template-id>
   ```

   Expected: it creates another workspace when the tenant is below
   `maxWorkspacesPerTenant` and a GPU is available, or queues it when GPUs are
   busy. Remove the extra workspace before continuing.

3. Verify ownership and management labels in the tenant's rootless runtime:

   ```bash
   podman inspect -f '{{ index .Config.Labels "io.mtcode.managed" }}' "$WS_NAME"
   podman inspect -f '{{ index .Config.Labels "io.mtcode.template" }}' "$WS_NAME"
   ```

   Expected: `true` and the selected template ID.

### 4. Private-home and read-only dataset mounts

1. On the SSH host, create a tenant-owned marker:

   ```bash
   printf 'from-host\n' > ~/container-manual-test.txt
   ```

2. Read it inside the container, then write a second marker from the container:

   ```bash
   mt-container exec "$WS_ID" -- cat /workspace/container-manual-test.txt
   mt-container exec "$WS_ID" -- sh -c 'printf "from-container\n" > /workspace/container-wrote.txt'
   cat ~/container-wrote.txt
   ```

   Expected: both directions succeed, proving that the private home is mounted
   read-write at `/workspace`.

3. Test one configured dataset, substituting its absolute path:

   ```bash
   mt-container exec "$WS_ID" -- test -r <absolute-dataset-path>
   mt-container exec "$WS_ID" -- touch <absolute-dataset-path>/mtgpu-write-should-fail
   ```

   Expected: the read test succeeds and `touch` fails with a read-only or
   permission error. Confirm that the test file was not created.

### 5. Lifecycle, shell, and non-interactive execution

Run each operation from the SSH host:

```bash
mt-container status "$WS_ID"
mt-container exec "$WS_ID" -- cat /etc/os-release
mt-container shell "$WS_ID"
# Exit the container shell with `exit`, then continue:
mt-container stop "$WS_ID"
mt-container status "$WS_ID"
mt-container start "$WS_ID"
mt-container status "$WS_ID"
```

Expected: `exec` runs without an interactive shell; `shell` opens `/bin/bash`;
state moves from `running` to `exited` and back to `running`. Inside the shell,
`mt-container` is expected to be absent. A prompt such as `root@<id>` is root
inside a rootless container, not host root.

### 6. GPU access

Use a CUDA/NVIDIA template that contains `nvidia-smi`, then run:

```bash
mt-container exec "$WS_ID" -- nvidia-smi
```

Expected: the command lists the host driver and the GPUs assigned by the
runtime. For a framework image, also use its native check, for example:

```bash
mt-container exec "$WS_ID" -- python -c 'import torch; print(torch.cuda.is_available()); print(torch.cuda.device_count())'
```

Expected: CUDA availability is `True` and the current implementation reports
all exposed GPUs. A generic Ubuntu/Python image may not contain `nvidia-smi` or
GPU framework libraries even though GPU devices were attached.

### 6a. Per-GPU selection and reservation

GPU requests are chosen per workspace. Omitting `--gpu-count`, `--gpus`, and
`--gpu-indexes` requests no GPU visibility. Users may select `--gpu-count N`
to reserve any N available GPUs up to the detected GPU count, or
`--gpu-indexes 0,1` to reserve specific GPU indexes. On
Linux/NVIDIA hosts, admission also excludes devices that currently have native
(non-container) compute processes. Assignments persist in
`<ROOT_FOLDER>/.mtgpu/gpu-reservations.json` and are released on workspace
removal or account cleanup.

1. Create a workspace with one requested GPU and confirm only the reserved
   device is exposed:

   ```bash
   mt-container create <template-id> --gpu-count 1
   mt-container list
   WS_ID=<new-id-from-list>
   mt-container status "$WS_ID"     # shows the assigned GPU index/indices
   mt-container exec "$WS_ID" -- nvidia-smi -L
   ```

   Expected: `nvidia-smi -L` lists exactly the requested GPU count (not all host
   GPUs), and the GPU Panel workspace line shows `... · GPU <index>`.

2. Confirm the reservation is recorded on the host:

   ```bash
   sudo jq . /home/zezhen/GPU_SERVER_DISK/.mtgpu/gpu-reservations.json
   ```

   Expected: an object mapping each generated container name to its reserved
   indices, for example `{"mtgpu-ws-64656d6f-1":[0]}` for account `demo`.

3. Exclusivity: while the first workspace holds every GPU, have a second tenant
   create a workspace.

   Expected: the second workspace enters the FIFO queue (no partial allocation
   or oversubscription) and starts after the first releases its GPU. On a
   single-GPU host, omit GPU options on the second workspace to create a
   CPU-only workspace without GPU visibility.

4. Native-process admission: remove all workspaces, start a native CUDA workload
   on GPU 0, and then create a workspace while the workload is running:

   ```bash
   CUDA_VISIBLE_DEVICES=0 python <your-cuda-workload.py> &
   nvidia-smi --query-compute-apps=gpu_uuid,pid,process_name --format=csv
   mt-container create <template-id>
   mt-container list
   ```

   Expected on a multi-GPU host: the workspace is assigned another free GPU.
   Expected when fewer than the requested devices remain: the workspace is
   queued and no reservation is written until enough GPUs are free. Processes
   already running inside managed containers are identified through `/proc`
   cgroups and are not double-counted as native workloads.

5. Stop the workspace, start a native workload on its assigned GPU, and run:

   ```bash
   mt-container start "$WS_ID"
   ```

   Expected: start fails with "an assigned GPU is occupied by a native process".
   Stop the native workload and confirm `mt-container start "$WS_ID"` then
   succeeds.

   Linux provisioning also installs a best-effort native shell guard: new
   interactive shells run `mt-container native-env` when
   `CUDA_VISIBLE_DEVICES` is not already set, hiding GPUs currently reserved by
   managed containers from ordinary CUDA programs. Verify from a fresh SSH shell:

   ```bash
   mt-container native-env
   eval "$(mt-container native-env)"
   echo "$CUDA_VISIBLE_DEVICES"
   echo "$MTGPU_CONTAINER_RESERVED_GPUS"
   echo "$MTGPU_CONTAINER_NATIVE_BUSY_GPUS"
   echo "$MTGPU_CONTAINER_NATIVE_GPU_CONFLICTS"
   ```

   Expected: `mt-container native-env` prints shell exports. `eval` applies
   them to the current shell, after which reserved container GPUs are omitted
   from `CUDA_VISIBLE_DEVICES` and listed in `MTGPU_CONTAINER_RESERVED_GPUS`.
   `MTGPU_CONTAINER_NATIVE_BUSY_GPUS` lists native-process GPU usage, and
   `MTGPU_CONTAINER_NATIVE_GPU_CONFLICTS` is normally empty; a non-empty value
   means a native process is currently using a GPU reserved by a managed
   container. New SSH shells get the managed profile hook automatically. This
   prevents accidental conflicts in normal shells and helps diagnose deliberate
   conflicts; it is not a hard security boundary because a user can deliberately
   override the variable.

6. Release: remove the first workspace and confirm the device frees up:

   ```bash
   mt-container remove "$WS_ID"
   sudo jq . /home/zezhen/GPU_SERVER_DISK/.mtgpu/gpu-reservations.json
   ```

   Expected: that generated container-name entry is gone and a subsequent
   create can reuse the freed GPU.

### 6b. Historical GPU usage

On Linux/NVIDIA hosts, a background sampler records per-GPU utilization/memory,
per-account native versus container GPU-memory/process totals, and estimated
per-account GPU utilization into a bounded ring buffer at
`<ROOT_FOLDER>/.mtgpu/usage-history.json`, surfaced in the GPU Panel. Query
responses contain only the authenticated tenant's account attribution. The same
history includes per-tenant running-container CPU/RAM/PID aggregates sampled
from the coordinated container broker. Per-account GPU utilization is marked
estimated because exact process-level availability depends on the host and
driver: MTGPU uses `nvidia-smi pmon` per-process SM samples when available, and
otherwise splits NVIDIA's per-GPU sample across active GPU processes for
attribution.

1. Run a GPU workload (native or in a workspace) for a few minutes, then inspect
   the on-disk history:

   ```bash
   sudo jq '.[-3:]' /home/zezhen/GPU_SERVER_DISK/.mtgpu/usage-history.json
   ```

   Expected: a growing array of `{ts, gpus:[{name, util, memUsedBytes,
   memTotalBytes, accounts:[...]}]}` samples. Account entries separate
   `nativeMemUsedBytes`/`nativeProcesses` from
   `containerMemUsedBytes`/`containerProcesses` and include
   `estimatedGpuUtil` with `estimatedGpuUtilSource`
   (`nvidia-smi-pmon-sm` when per-process SM samples are available, otherwise
   `split-by-active-process-count`); the buffer is capped (oldest samples drop)
   and survives a server restart.

2. In the GPU Panel, select the server. Under **Live Usage** a per-GPU sparkline
   row appears (e.g. `NVIDIA … : ▁▃▆█▅▂ peak 87% (30m) yours: native 1.0 GiB,
   container 2.0 GiB`), refreshing about every 30 seconds.

   Expected: the sparkline tracks the recent workload and the peak matches the
   observed `nvidia-smi` utilization. The memory suffix separates the requesting
   tenant's current native and container GPU allocations and shows estimated GPU
   utilization when attribution data is present. When managed containers are
   running, a **Containers** row shows historical CPU, latest RAM, and PID count
   for the authenticated tenant.

3. Privacy check: query history as two different tenants. Each response's
   `accounts` arrays contain only the requesting account, while the root-owned
   mode-`0600` on-disk history retains all aggregates for administration.

### 7. CPU, memory, and PID limits

1. Remove and recreate the workspace after changing limits; policies are
   applied only during creation:

   ```bash
   mt-container remove "$WS_ID"
   mt-container create <template-id>
   mt-container list
   WS_ID=<new-id-from-list>
   WS_NAME=$(mt-container status "$WS_ID" | jq -r .name)
   ```

2. Inspect the runtime configuration:

   ```bash
   podman inspect "$WS_NAME" | jq '.[0].HostConfig | {NanoCpus,Memory,PidsLimit}'
   ```

   Expected for the example policy: approximately 1.5 CPUs, 2147483648 bytes of
   memory, and 256 PIDs. Podman and Docker versions may expose CPU fields under
   slightly different names; `mt-container status` must still report the
   configured values.

3. Generate a live resource sample through the coordinated CLI:

   ```bash
   mt-container stats <id>
   ```

   Expected: CPU percent, current RAM usage/percent, and PID count are reported.
   In the GPU Panel, the **Containers** row shows the authenticated tenant's
   aggregate CPU/RAM/PID usage, and **Container Workspace → Show Resources**
   shows the selected workspace. Stopped workspaces report stats unavailable.

### 8. Network policy

1. With `networkPolicy: "disabled"`, remove and recreate the workspace.
2. Confirm the runtime network mode:

   ```bash
   podman inspect "$WS_NAME" | jq -r '.[0].HostConfig.NetworkMode'
   ```

   Expected: `none`. Network clients inside a suitably equipped image must fail
   to reach external hosts.

3. Change the policy to `egress`, restart the server, reconnect, remove, and
   recreate the workspace. Repeat the inspection and an image-appropriate
   network request such as `curl -I https://example.com`.

   Expected: the container uses the rootless runtime's normal network and the
   request succeeds when host DNS/firewall policy permits it. Image pulling is
   performed by the host runtime and is not proof of in-container egress.

### 8a. Published ports

Workspace-selected container ports are published with auto-assigned host ports,
and only when `networkPolicy` is not `disabled` (on rootless engines publishing
a port requires a network namespace, which also permits egress). The default
bind host is `127.0.0.1`; administrators may explicitly set
`CONTAINERS.publishHost: "0.0.0.0"` to expose the host port beyond loopback.
Users select ports per workspace with `--publish-ports 8888,6006`. Omitting the
option, or using `--publish-ports none`, publishes no ports.

1. Configure egress, then restart, reconnect, and recreate the workspace:

   ```json
   "CONTAINERS": {
     "networkPolicy": "egress",
     "publishHost": "127.0.0.1"
   }
   ```

   Then create a workspace with an explicit published port:

   ```bash
   mt-container create --template pytorch-2.4-cuda12.4 --publish-ports 8888
   # or publish multiple container listening ports:
   mt-container create --template ml-py312 --publish-ports 8888,6006,7860
   # or explicitly publish nothing:
   mt-container create --template ml-py312 --publish-ports none
   ```

2. Start a listener inside the container on the selected port and read the mapping:

   ```bash
   mt-container exec "$WS_ID" -- python -m http.server 8888 &
   mt-container status "$WS_ID" # "ports": [{containerPort 8888, host 127.0.0.1, hostPort <auto>}]
   podman port "$WS_NAME"           # 8888/tcp -> 127.0.0.1:<auto>
   ```

   Expected: the port is bound to `127.0.0.1` on the GPU host and the mapping
   appears in `mt-container status` (the GPU Panel renders the forwarded link
   in its **Ports:** row — see step 4).

3. Inspect the admin route manifest:

   ```bash
   sudo jq . /home/zezhen/GPU_SERVER_DISK/.mtgpu/container-port-routes.json
   ```

   Expected: the manifest contains a `routes` entry for the running workspace
   with the tenant account, workspace id/name, container port, protocol, bind
   host, auto-assigned host port, and URL. This is the stable source of truth a
   future reverse proxy / DirectLink router can consume; it is mode `0600`
   because it includes tenant/workspace metadata. Stop or remove the workspace
   and confirm the route disappears.

4. Reach it from the host loopback:

   ```bash
   curl -sS http://127.0.0.1:<auto>/ | head        # on the GPU host
   ```

   Expected: the service responds on the host loopback; it is not reachable on
   the host's public interface or by other tenants.

5. Client auto-forwarding (from a local panel, not a remote window): the GPU
   Panel shows a **Ports:** row with a clickable `8888 → 127.0.0.1:<clientPort>`
   link; the extension starts a managed companion `ssh -L` connection through
   the server's established SSH endpoint so the service opens on the client's
   localhost.

   Expected: clicking the link opens the service locally; the forward is removed
   when the workspace is stopped/removed or another server is selected.

6. Set `networkPolicy: "disabled"`, recreate, and confirm no ports are published
   (`mt-container status` reports an empty `ports` list) even when the workspace
   requests ports; the route manifest has an empty `routes` array.

7. Public bind test (disposable host only): set `publishHost: "0.0.0.0"`,
   restart, reconnect, remove/recreate the workspace, start the listener again,
   and inspect the mapping.

   ```bash
   mt-container status "$WS_ID" # "ports": [{containerPort 8888, host 0.0.0.0, hostPort <auto>}]
   podman port "$WS_NAME"           # 8888/tcp -> 0.0.0.0:<auto>
   ```

   Expected: the GPU Panel shows a direct public link using the server address
   instead of creating a client-side SSH forward. Firewall, cloud security
   groups, NAT, and site policy still control whether remote clients can reach
   that port.

#### Routed exposure options

The route manifest is a foundation for several future routed-exposure designs.
The central MTCode website should usually be the **control plane** rather than
the default data plane, because proxying every notebook, TensorBoard, Gradio,
file download, or model artifact through `mtcodeai.com` would require
significant central bandwidth.

Possible designs:

- **Control-plane only through `mtcodeai.com`, data-plane direct**:
  `mtcodeai.com` authenticates the user and issues a short-lived route token,
  while the browser/client connects directly to the GPU server or DirectLink
  endpoint for the actual container traffic.
- **Per-GPU-server reverse proxy**: each GPU server runs a local authenticated
  proxy that reads `.mtgpu/container-port-routes.json` and forwards
  `https://<gpu-server>/...` requests to the local `127.0.0.1:<hostPort>`
  mappings. This keeps data bandwidth on the GPU host instead of the central
  website.
- **DirectLink relay only as fallback**: try direct/peer routing first; use a
  relay only when NAT, firewall, or site policy prevents a direct connection.
  This keeps central relay bandwidth proportional to the hard cases instead of
  every port session.

Current implementation: direct publish, client SSH forwarding, and the route
manifest are implemented. MTGPU does not yet run an authenticated HTTPS
proxy/router or central relay for container ports.

### 9. Process inspection, signaling, and logs

1. Start a disposable background process and capture its in-container PID:

   ```bash
   PID="$(mt-container exec "$WS_ID" -- sh -c 'sleep 600 >/dev/null 2>&1 & echo $!')"
   printf 'test PID=%s\n' "$PID"
   mt-container ps "$WS_ID"
   ```

   Expected: the process table contains that PID and its `sleep 600` command.

2. Signal it and inspect the table again:

   ```bash
   mt-container kill "$WS_ID" "$PID" TERM
   mt-container ps "$WS_ID"
   ```

   Expected: signaling succeeds and the process disappears. Repeat through
   **Container Workspace → Manage Processes** in the GPU Panel if desired. Use
   `KILL` only when graceful signals do not work.

3. Query recent main-process output:

   ```bash
   mt-container logs "$WS_ID"
   ```

   Expected: logs are printed, or the CLI reports that no container output is
   available. The managed `sleep infinity` main process normally produces no
   output; output from `exec` is returned directly and is not necessarily part
   of the container's main-process log. The panel's **Show Logs** action must
   show the same result without failing.

### 10. Persistent named volumes

1. Confirm the provisioned volume policy:

   ```bash
   mt-container volumes
   ```

   Expected: `manual-test-cache` maps to `/mtgpu/cache`.

2. Write a marker, remove the workspace, recreate it, and read the marker:

   ```bash
   mt-container exec "$WS_ID" -- sh -c 'printf "persistent\n" > /mtgpu/cache/manual-test.txt'
   mt-container remove "$WS_ID"
   mt-container create <template-id>
   mt-container list
   WS_ID=<new-id-from-list>
   mt-container exec "$WS_ID" -- cat /mtgpu/cache/manual-test.txt
   ```

   Expected: `persistent` remains because workspace removal does not delete
   administrator-configured named volumes. The GPU Panel's **Show Volumes**
   action must list the same name and mount path.

### 10a. Detached workspace jobs

`mt-container job-run` starts a non-interactive command in an existing
running workspace and records status/logs below
`~/.mtgpu/container-jobs/<job-id>`. This is separate from the interactive
workspace admission queue: the job uses the workspace's assigned resources. If
the target workspace is currently queued, add `--wait` before `--` to wait for
the workspace admission scheduler to start it, then launch the detached job; add
`--wait-timeout SECONDS` to cap that queue wait. Use `--timeout SECONDS` before
`--` to cap a job's wall-clock runtime without requiring `timeout` or other
helper packages inside the container image. Use `--retries N` to retry failed
exit codes; cancelled and timed-out jobs are not retried.
`mt-container job-submit` is the first job-first scheduling bridge: it creates
or queues a new workspace request with GPU admission options, waits for that
workspace to become running, then starts the same detached job record in it.
Completed job records can be pruned manually with `job-prune`; pruning never
removes `starting` or `running` jobs. The server also prunes old terminal job
records during the hourly cleanup sweep according to
`JOB_RETENTION_DAYS` (default 30; `0` disables automatic pruning). The same
window removes unused completed `.tar` files from `CONTAINERS.sharedImageDir`,
except images pinned by any cache-sharing member's
`CONTAINERS.prePullImages`; successful cache loads and cluster transfers
refresh a tar's retention clock. Members may use different pre-pull lists:
atomic, checksummed `.pins` sidecars retain the union of their renewable
per-server leases and are rebuilt with a grace period if missing or damaged.

1. Start a short job:

   ```bash
   mt-container job-run "$WS_ID" -- sh -c 'echo hello; sleep 2; echo done'
   mt-container jobs
   ```

   Expected: the command prints a job ID and `jobs` shows `starting` or
   `running`, then `succeeded`.

2. Inspect status and logs:

   ```bash
   JOB_ID=<job-id-from-output>
   mt-container job-status "$JOB_ID"
   mt-container job-logs "$JOB_ID"
   ```

   Expected: status shows the workspace ID, command, final state, and exit code;
   logs contain `hello` and `done`.

3. Cancel a longer job:

   ```bash
   mt-container job-run "$WS_ID" -- sh -c 'sleep 600'
   JOB_ID=<job-id-from-output>
   mt-container job-cancel "$JOB_ID"
   mt-container job-status "$JOB_ID"
   ```

   Expected: the job state becomes `cancelled`. The job files remain under the
   tenant home for later inspection and are removed with normal tenant-home
   cleanup.

4. Enforce a job time limit:

   ```bash
   mt-container job-run "$WS_ID" --timeout 5 -- sh -c 'echo start; sleep 60; echo never'
   JOB_ID=<job-id-from-output>
   sleep 8
   mt-container job-status "$JOB_ID"
   mt-container job-logs "$JOB_ID"
   ```

   Expected: status reports `timed-out`, includes `Timeout: 5 seconds`, and the
   logs contain `start` but not `never`.

5. Retry a transient failure:

   ```bash
   mt-container job-run "$WS_ID" --retries 1 -- sh -c 'test -f /workspace/.mtgpu-retry-ok || { touch /workspace/.mtgpu-retry-ok; echo first-failed; exit 7; }; echo retried-ok'
   JOB_ID=<job-id-from-output>
   sleep 4
   mt-container job-status "$JOB_ID"
   mt-container job-logs "$JOB_ID"
   rm -f ~/.mtgpu-retry-ok
   ```

   Expected: status reports `succeeded`, includes `Retries: 1` and
   `Attempt: 2`, and the logs contain both `first-failed` and `retried-ok`.

6. Prune old completed job records:

   ```bash
   mt-container job-prune --older-than 0 --dry-run
   mt-container job-prune --older-than 0
   mt-container jobs
   ```

   Expected: only terminal jobs (`succeeded`, `failed`, `cancelled`,
   `timed-out`, or `failed-to-start`) are reported/removed. Jobs still
   `starting` or `running` remain.

7. Submit a job to a queued workspace:

   ```bash
   # On a busy 1-GPU host, create a second workspace so it queues.
   mt-container create <template-id>
   QUEUED_WS_ID=<queued-workspace-id-from-list>
   mt-container job-run "$QUEUED_WS_ID" --wait --wait-timeout 600 -- sh -c 'echo admitted; nvidia-smi -L'
   ```

   Expected: the command prints that it is waiting while the workspace is
   queued. After the admission scheduler starts that workspace, the job starts
   and `job-status`/`job-logs` show the normal detached-job record. If the
   workspace fails or the wait timeout expires, no running job is left behind.

8. Test server-side job retention automation:

   ```bash
   # In config/config.json, set JOB_RETENTION_DAYS to 1, restart the server,
   # then create a disposable terminal job record whose ID timestamp is older
   # than one day, or wait for an old completed job record to age past the limit.
   ```

   Expected: the hourly cleanup loop logs that old detached container job records
   were pruned. Running/starting job records are preserved.

9. Submit a job-first workspace request:

   ```bash
   mt-container job-submit pytorch-2.4-cuda12.4 --gpu-count 1 --priority 10 --timeout 300 --retries 1 -- sh -c 'echo batch-start; nvidia-smi -L'
   mt-container list
   mt-container jobs
   ```

   Expected: if a GPU is free, the command creates a workspace and starts the
   detached job immediately. If GPUs are busy, the workspace queues with the
   requested priority; the CLI waits until admission starts the workspace, then
   starts the detached job. If `--wait-timeout SECONDS` is supplied and expires,
   the workspace remains queued and no job is started.

   Jobs run from `/workspace` by default, which is the container mount of the
   tenant home. For project commands, wrap the real command with `sh -c` and
   `cd` into the project directory first:

   ```bash
   mt-container job-submit ml-py312 --gpu-count 1 -- \
     sh -c 'cd /workspace/Projects/classifier-gan-copy && python train.py --help'
   mt-container jobs
   JOB_ID=<job-id-from-output>
   mt-container job-logs "$JOB_ID"
   ```

   Expected: `job-logs` shows the command output from inside the project
   directory. Replace `python train.py --help` with the actual training command,
   for example `cd /workspace/Projects/<project> && python train.py ...`.

### 11. Administrator-gated custom OCI images

1. With `allowCustomImages: true`, remove the current workspace and create a
   public custom image:

   ```bash
   mt-container remove "$WS_ID"
   mt-container create --image docker.io/library/ubuntu:24.04
   mt-container list
   WS_ID=<new-id-from-list>
   WS_NAME=$(mt-container status "$WS_ID" | jq -r .name)
   podman inspect -f '{{ index .Config.Labels "io.mtcode.image" }}' "$WS_NAME"
   ```

   Expected: `mt-container status "$WS_ID"` returns JSON with
   `"templateId": "custom"`, and the label prints the exact image reference. In
   the GPU Panel, **Container Workspace → Use Custom
   Image...** must provide the equivalent flow.

2. Negative policy test: set `allowCustomImages` to `false`, restart, reconnect,
   remove the workspace, then run:

   ```bash
   mt-container create --image docker.io/library/ubuntu:24.04
   ```

   Expected: the CLI reports that custom images are disabled and the panel no
   longer offers **Use Custom Image...**. Restore the intended policy afterward.

3. Private images are supported two ways (see §11a). Never paste credentials
   into `config.json`, image references, logs, or this manual.

### 11a. Registry credentials for private images

Two paths cover private registries; the server never stores plaintext passwords
in `config.json`.

Admin-shared images (credentials stay confidential):

1. As an administrator on the host, create an auth file and configure the share:

   ```bash
   podman login --authfile /etc/mtgpu/registry-auth.json ghcr.io   # prompts; not logged
   ```

   ```json
   "CONTAINERS": {
     "registryAuthfile": "/etc/mtgpu/registry-auth.json",
     "sharedImageDir": "/home/zezhen/GPU_SERVER_DISK/.mtgpu/shared-images",
     "prePullImages": ["ghcr.io/yourorg/private-ml:py312"]
   }
   ```

2. Restart the server. A background pass pulls each `prePullImages` entry with the
   admin auth file and exports it under `sharedImageDir`:

   ```bash
   ls -l /home/zezhen/GPU_SERVER_DISK/.mtgpu/shared-images   # ghcr.io_yourorg_private-ml_py312.tar, world-readable
   ```

   Expected log: `[container] shared image ready: ghcr.io/yourorg/private-ml:py312`.

3. As a tenant, create a workspace using that image (custom image or a template
   whose `image` matches). It must succeed with **no** tenant `podman login`:
   the workspace creation loads the shared tarball into the tenant store. Confirm
   the registry credentials never reached the tenant (the tenant's
   `~/.config/containers/auth.json` is absent or unchanged).

Per-tenant credentials (a tenant's own private registry):

4. In the tenant SSH terminal, log the tenant's rootless engine in, then create a
   workspace from an image only that tenant can pull:

   ```bash
   podman login ghcr.io                       # stores ~/.config/containers/auth.json
   mt-container create <template-or-custom-id>
   ```

   Expected: the workspace pull (run as the tenant by the server) uses the
   tenant's own auth file; other tenants without that login cannot pull the same
   image.

### 11b. Runtime secret-file injection

The administrator lists secret files; matching tenants receive them bind-mounted
read-only inside the workspace. Secret values never appear in `config.json`, the
panel, status, or logs — only the file path on the host and the in-container
target. Because a rootless container runs as the tenant, a mounted secret is by
nature readable by that tenant; use this for secrets the tenant is trusted with,
not to hide values from them.

1. As administrator, create a secret file readable only by the server, and
   declare it for `demo` on a template:

   ```bash
   install -m 600 /dev/stdin /etc/mtgpu/secrets/hf_token <<<'hf_xxx_REDACTED'
   ```

   ```json
   "CONTAINERS": {
     "runtimeSecrets": [
       {
         "name": "hf_token",
         "source": "/etc/mtgpu/secrets/hf_token",
         "target": "/run/secrets/hf_token",
         "accounts": ["demo"],
         "templates": ["ml-py312"]
       }
     ]
   }
   ```

2. Restart the server (a startup warning is logged if `source` is unreadable),
   then as `demo` create the matching workspace and read the secret inside it:

   ```bash
   mt-container create ml-py312
   mt-container list
   WS_ID=<new-id-from-list>
   mt-container status "$WS_ID"      # "secrets":[{name hf_token, target /run/secrets/hf_token}]
   mt-container exec "$WS_ID" -- cat /run/secrets/hf_token
   ```

   Expected: the value is present at the target; `mt-container status` (and the
   GPU Panel workspace line, `... · secrets hf_token`) lists the name/target but
   never the value. A tenant or template not listed receives no secret.

3. Confirm the staged copy is tenant-owned, mode 0400, and removed with the
   workspace:

   ```bash
   sudo ls -l "/home/zezhen/GPU_SERVER_DISK/.mtgpu/runtime-secrets/$WS_NAME/"
   mt-container remove "$WS_ID"
   sudo ls "/home/zezhen/GPU_SERVER_DISK/.mtgpu/runtime-secrets/$WS_NAME/" 2>&1   # gone
   ```

   Expected: `-r-------- demo demo hf_token` while the workspace exists, and the
   staging directory is deleted on removal (and on retention cleanup).

4. Build-time use (devcontainer Dockerfile): declare the secret for the
   `devcontainer` template and consume it during the build without leaking it
   into a layer:

   ```bash
   # config: add "devcontainer" to the secret's templates array, then as demo:
   cat > ~/.devcontainer/Dockerfile <<'DOCKERFILE'
   FROM docker.io/library/python:3.12-slim
   RUN --mount=type=secret,id=hf_token \
       test -s /run/secrets/hf_token && echo "secret visible at build" > /built-with-secret
   DOCKERFILE
   cat > ~/.devcontainer/devcontainer.json <<'JSON'
   { "name": "secret build", "build": { "dockerfile": "Dockerfile", "context": "." },
     "workspaceFolder": "/workspace" }
   JSON
   mt-container remove "$WS_ID"
   mt-container create --devcontainer .devcontainer/devcontainer.json
   mt-container list
   WS_ID=<new-id-from-list>
   mt-container exec "$WS_ID" -- cat /built-with-secret
   podman history --no-trunc localhost/mtgpu/devcontainer-* | grep -c hf_      # 0
   ```

   Expected: the build succeeds using the mounted secret, `/built-with-secret`
   exists, and the value never appears in image history/layers (podman passes it
   via `--secret`). On a Docker host this requires BuildKit.

### 12. Image, Dockerfile, and Compose `devcontainer.json`

1. In the tenant's SSH home, create a configuration:

   ```bash
   mkdir -p ~/.devcontainer
   cat > ~/.devcontainer/devcontainer.json <<'JSON'
   {
     "name": "Manual test",
     "image": "docker.io/library/ubuntu:24.04",
     "workspaceFolder": "/workspace",
     "customizations": {
       "vscode": {
         "settings": {
           "terminal.integrated.defaultProfile.linux": "bash"
         }
       }
     }
   }
   JSON
   ```

2. Remove the current workspace and create from that file:

   ```bash
   mt-container remove "$WS_ID"
   mt-container create --devcontainer .devcontainer/devcontainer.json
   mt-container list
   WS_ID=<new-id-from-list>
   WS_NAME=$(mt-container status "$WS_ID" | jq -r .name)
   podman inspect -f '{{ index .Config.Labels "io.mtcode.devcontainer" }}' "$WS_NAME"
   podman inspect -f '{{ index .Config.Labels "devcontainer.metadata" }}' "$WS_NAME" | jq .
   ```

   Expected: `mt-container status "$WS_ID"` returns JSON with
   `"templateId": "devcontainer"`; the first label contains the canonical path
   below the tenant home; the metadata preserves the terminal setting and adds
   `files.dialog.defaultPath: /workspace`.

3. Repeat from **Container Workspace → Use devcontainer.json...** in the GPU
   Panel. Enter `.devcontainer/devcontainer.json` and confirm the same result.

4. Test a Dockerfile build. Replace the configuration and add a Dockerfile:

   ```bash
   cat > ~/.devcontainer/Dockerfile <<'DOCKERFILE'
   FROM docker.io/library/ubuntu:24.04
   RUN printf 'first-build\n' > /mtgpu-build-version
   DOCKERFILE
   cat > ~/.devcontainer/devcontainer.json <<'JSON'
   {
     "name": "Built manual test",
     "build": {
       "dockerfile": "Dockerfile",
       "context": ".",
       "args": {}
     },
     "workspaceFolder": "/workspace"
   }
   JSON
   mt-container remove "$WS_ID"
   mt-container create --devcontainer .devcontainer/devcontainer.json
   mt-container list
   WS_ID=<new-id-from-list>
   mt-container exec "$WS_ID" -- cat /mtgpu-build-version
   ```

   Expected: the tenant's rootless engine builds a private
   `localhost/mtgpu/devcontainer-...:latest` image and the command prints
   `first-build`.

5. Change `first-build` to `second-build` in the Dockerfile and rebuild:

   ```bash
   sed -i 's/first-build/second-build/' ~/.devcontainer/Dockerfile
   mt-container rebuild "$WS_ID"
   mt-container list
   WS_ID=<rebuilt-id-from-list>
   mt-container exec "$WS_ID" -- cat /mtgpu-build-version
   ```

   Expected: the Dockerfile is rebuilt, the managed container is replaced, and
   the command prints `second-build`. Files below `/workspace` and configured
   named volumes remain. The GPU Panel exposes the same operation as
   **Container Workspace → Rebuild**. A failed build leaves the old container
   removed, so fix the build and run the create command again.

6. Test a constrained Docker Compose devcontainer. This requires `podman compose`
   or `docker compose` to be available for the tenant account. The
   devcontainer must declare exactly one primary `service`; MTGPU injects the
   managed container name, labels, `/workspace` mount, dataset mounts,
   runtime-secret mounts, GPU assignment, resource limits, and network/port
   policy through a generated override file.

   ```bash
   cat > ~/.devcontainer/compose.yml <<'YAML'
   services:
     app:
       image: docker.io/library/ubuntu:24.04
       command: sleep infinity
   YAML
   cat > ~/.devcontainer/devcontainer.json <<'JSON'
   {
     "name": "Compose manual test",
     "dockerComposeFile": "compose.yml",
     "service": "app",
     "workspaceFolder": "/workspace"
   }
   JSON
   mt-container remove "$WS_ID"
   mt-container create --devcontainer .devcontainer/devcontainer.json
   mt-container list
   WS_ID=<new-id-from-list>
   WS_NAME=$(mt-container status "$WS_ID" | jq -r .name)
   mt-container exec "$WS_ID" -- pwd
   podman inspect -f '{{ index .Config.Labels "io.mtcode.managed" }}' "$WS_NAME"
   podman inspect -f '{{ index .Config.Labels "io.mtcode.devcontainer" }}' "$WS_NAME"
   ```

   Expected: the command prints `/workspace`, the labels print `true` and the
   canonical devcontainer path, and stop/start/remove/status/logs/processes act
   on the selected Compose service. Sidecar services are intentionally out of
   scope for this first implementation; if present, `remove` runs Compose
   `down --remove-orphans` for the generated project.

7. Security and unsupported-form tests:

   ```bash
   mt-container remove "$WS_ID"
   mt-container create --devcontainer /etc/passwd
   ```

   Expected: paths outside the tenant home are rejected. Dockerfiles, build
   contexts, and Compose files escaping through `..` or symlinks are also
   rejected. A configuration using `runArgs`, mounts, privileged mode,
   lifecycle commands, or other unsupported properties must fail explicitly.
   Compose configurations must include both `dockerComposeFile` and `service`,
   must not also specify top-level `image`/`build`, and must keep
   `workspaceFolder` at `/workspace`. JSON comments are accepted; trailing
   commas are not yet accepted. Do not put registry
   passwords or secret values in `build.args`, because command-line build
   arguments are not a secrets-management mechanism.

### 13. GPU Panel and Remote-SSH parity

1. Create a workspace from the GPU Panel, then open the tenant SSH terminal and
   run `mt-container status`.
2. Stop or start it with the CLI and refresh/reselect **Container Workspace** in
   the panel.
3. Use **Open Terminal** from both a local MTGPU window and a Remote-SSH window.

Expected: panel and CLI always see the same workspace IDs and generated runtime
names; lifecycle changes are not duplicated or cached as a second workspace. A
local terminal reaches the host through the existing SSH tunnel, while a
Remote-SSH terminal executes directly on that host.

### 14. Dev Containers attachment and `/workspace` picker default

This test requires official Microsoft VS Code with
`ms-vscode-remote.remote-containers`. MTCode Studio intentionally shows the
attachment action as unavailable because the proprietary extension is not
distributed through Open VSX.

1. Remove and recreate the workspace so it receives current Dev Container
   metadata.
2. Verify the standard metadata label:

   ```bash
   podman inspect -f '{{ index .Config.Labels "devcontainer.metadata" }}' "$WS_NAME" | jq .
   ```

   Expected: `customizations.vscode.settings.files.dialog.defaultPath` is
   `/workspace`.

3. In the local official VS Code window, select **Container Workspace → Attach
   with Dev Containers** once. MTGPU records a five-minute, one-use handoff,
   opens the matching Remote-SSH authority, and passes the full runtime
   container ID to Dev Containers.

   Expected: the Remote-SSH window automatically transitions into the
   selected Dev Container without asking the user to select a container.
   If the server predates container-ID reporting, reconnect after upgrading it.

4. In the resulting Dev Container window, run **File: Open Folder...**.

   Expected: the picker starts at `/workspace`. Open a project below that path
   and confirm edits appear in the tenant home on the SSH host. Existing
   containers created before the metadata feature must be removed and recreated.

### 15. Workspace metadata, removal, and tenant-retention cleanup

1. While a workspace exists, inspect its tenant metadata:

   ```bash
   sudo jq . /home/zezhen/GPU_SERVER_DISK/.mtgpu/workspaces/demo.json
   ```

   Expected: it records every workspace ID, generated runtime name, GPU
   assignment, and template/custom selection. The runtime's `io.mtcode.image`
   label is authoritative for each exact image.

2. Verify ordinary removal preserves tenant data and named volumes:

   ```bash
   printf 'keep-me\n' > ~/removal-preserves-home.txt
   mt-container remove "$WS_ID"
   cat ~/removal-preserves-home.txt
   mt-container list
   ```

   Expected: the removed ID is absent from the list. If it was the tenant's
   final workspace, the registry contains an empty `workspaces` array.

   Expected: the home marker remains, the container is absent, and workspace
   metadata is removed.

3. Retention cleanup is destructive. Test it only with a disposable tenant and
   back up `ROOT_FOLDER/.mtgpu/users.json` first. Set a short nonzero
   `TENANT_RETENTION_DAYS`, stop all clients for longer than that interval, and
   restart or wait for the hourly sweep.

   Expected: the log shows container cleanup before account deletion; the
   disposable container, its configured named volumes, OS account, home, and
   registry entry are removed. If container cleanup fails, account deletion is
   deferred and the error is logged. Never run this test against a real user's
   account.

### 15a. Reservation/orphan reconciliation and image pruning

The server runs a periodic reconciliation pass (alongside the cleanup sweep, and
even when `TENANT_RETENTION_DAYS=0`) that releases GPU reservations whose
workspace no longer exists and prunes dangling images in each reachable tenant
store.

1. Create a workspace (reserves a GPU), then remove its container *out of band*
   to simulate a crash or manual teardown, leaving the reservation stale:

   ```bash
   mt-container create <template-id>
   sudo jq . /home/zezhen/GPU_SERVER_DISK/.mtgpu/gpu-reservations.json
   podman rm -f "$WS_NAME"                                                # bypass the broker
   ```

2. Trigger reconciliation (restart the server, or wait for the hourly pass).

   Expected: the server logs `reconciled container GPU reservations; released N
   stale reservation(s)`, and the account's entry is gone from
   `gpu-reservations.json` so the freed GPU is reusable.

3. Dangling-image pruning: build a `devcontainer.json` workspace twice (which
   produces an untagged intermediate image), then let a reconciliation pass run.

   ```bash
   podman images -f dangling=true        # before: lists untagged layers
   # after a reconciliation pass:
   podman images -f dangling=true        # empty; tagged template/devcontainer images remain
   ```

   Expected: only dangling (untagged) images are removed; pulled template and
   tagged devcontainer images are untouched.

4. Reservation recovery: with a workspace running, back up and remove
   `gpu-reservations.json`, then restart the server or trigger reconciliation.

   Expected: the file is recreated mode `0600` with the running workspace's
   exact `gpus` assignment from its tenant registry. The container is not
   rebound to a different device.

5. Corrupt-registry fail-safe: back up the tenant registry, replace it briefly
   with invalid JSON, and trigger reconciliation.

   Expected: the server logs that the registry is unreadable, preserves that
   tenant's reservations, and refuses destructive retention cleanup. Restore
   the registry backup before continuing.

### 15b. Multiple workspaces and the GPU admission queue

A tenant may hold up to `CONTAINERS.maxWorkspacesPerTenant` workspaces (default
4). Each workspace defaults to no GPU visibility. Users request GPUs with
`--gpu-count N` up to the detected GPU count or with `--gpu-indexes 0,1` for
specific device indexes. When the requested GPUs are busy a new workspace is
**queued** and a
server-wide priority scheduler starts it automatically when enough free up.
Higher `--priority` values start first, and workspaces with the same priority
remain FIFO. Workspaces are tracked per tenant in
`<ROOT_FOLDER>/.mtgpu/workspaces/<account>.json`; GPU reservations
(`gpu-reservations.json`) are keyed by generated container name
(`mtgpu-ws-<hex-encoded-account>-<id>`).
Use the GPU Panel's **Container Workspace** action or the `mt-container` CLI
(`list`/`create`/`start`/`stop`/...) — both share the same registry and queue.

1. On a 1-GPU host, create workspace A (1 GPU) from the panel → it runs. Create
   workspace B (1 GPU) → it is **queued (position 1)**. The panel's Containers
   row shows `… · 2 workspaces (1 running, 1 queued)` and the workspace picker
   lists both with state/queue.

   ```bash
   sudo jq -r '.workspaces[] | "\(.id) \(.state) gpus=\(.gpus)"' \
     /home/zezhen/GPU_SERVER_DISK/.mtgpu/workspaces/demo.json
   sudo jq . /home/zezhen/GPU_SERVER_DISK/.mtgpu/gpu-reservations.json
   # Example for demo: {"mtgpu-ws-64656d6f-1":[0]}
   ```

2. Remove or stop A. Within ~15s (or immediately on the remove/stop event) the
   scheduler launches B; its registry entry flips `queued`→`running` and it takes
   the freed GPU.

   If A was stopped rather than removed, starting A while B owns A's configured
   GPU must fail with "configured GPUs are currently unavailable". After B stops
   or is removed, A starts on its original device assignment; its existing
   writable container layer is preserved rather than silently binding a
   different GPU.

3. Cap: create workspaces up to the limit; the next create fails with
   "workspace limit reached (4 per tenant)".

4. Priority ordering: while all GPUs are busy, queue one normal-priority
   workspace and then a higher-priority workspace:

   ```bash
   mt-container create <template-id> --priority 0
   mt-container create <template-id> --priority 10
   mt-container list
   sudo jq -r '.workspaces[] | "\(.id) \(.state) priority=\(.priority)"' \
     /home/zezhen/GPU_SERVER_DISK/.mtgpu/workspaces/demo.json
   ```

   Expected: the priority-10 workspace shows an earlier queue position than the
   priority-0 workspace even though it was created later. After the running
   workspace releases its GPU, the priority-10 workspace starts first.

5. Multi-GPU queue head (multi-GPU host): request a 2-GPU workspace while only 1
   is free → it stays queued until 2 are free; a same-or-lower-priority 1-GPU
   request queued behind it also waits (the head of the priority-ordered queue
   is not skipped).

6. Per-workspace ops: start/stop/remove/terminal/logs/processes act on the chosen
   workspace only; client port auto-forwarding opens links for every running
   workspace and tears them down on stop/remove.

7. Retention cleanup removes **all** of a tenant's workspaces, reservations,
   staged secrets, and the registry file.

> Migration: an existing single `mtgpu-workspace` container from the previous
> design is not adopted into the registry; remove it once
> (`podman rm -f mtgpu-workspace`). Its old account-keyed reservation is dropped
> by reconciliation. A stopped multi-workspace entry created by an early queue
> build that has an empty `gpus` array must also be removed/recreated once; the
> server refuses to guess a different GPU for its existing container.

### 16. Final cleanup and result recording

1. Remove the test workspace and tenant markers:

   ```bash
   for id in $(mt-container list | awk 'NR > 1 && $1 ~ /^[0-9]+$/ {print $1}'); do
     mt-container remove "$id"
   done
   rm -f ~/container-manual-test.txt ~/container-wrote.txt \
     ~/removal-preserves-home.txt
   ```

2. Restore the production `CONTAINERS` policy, restart the server, and reconnect
   once so tenants receive the final manifest.
3. Record the server commit, client-module commit, extension commit, engine and
   NVIDIA driver versions, tested template/image, and pass/fail result for each
   numbered section. Do not mark a feature validated solely because its command
   returned zero; compare the observed state with every **Expected** statement.

## Container implementation status

Keep these lists synchronized with the implementation. When a roadmap feature
is completed and validated, move it from **Still missing** to **Implemented**
and update the relevant user-manual section.

### Implemented

- Optional rootless Podman/Docker backend
- GPU runtime detection
- Curated template selection and image pulling
- Multiple persistent workspaces per tenant (`CONTAINERS.maxWorkspacesPerTenant`,
  default 4), each defaulting to no GPU visibility and allowing per-workspace
  `--gpu-count N` or `--gpu-indexes 0,1` requests up to the detected GPU count,
  tracked in a
  per-tenant registry (`.mtgpu/workspaces/<account>.json`)
- GPU admission queue: when GPUs are busy, a workspace is created `queued` and a
  background server-wide priority scheduler auto-starts it when enough GPUs free
  up (on any stop/remove and on a periodic tick); higher `--priority` values
  start first, FIFO within the same priority
- `mt-container` CLI as a coordinated thin client: it connects to the server's
  local Unix control socket (`SO_PEERCRED` uid auth) and delegates workspace
  operations, sharing the same registry/reservations/queue as the GPU Panel
  (interactive shell/exec still run `podman exec` on the resolved container)
- Local broker hardening: only provisioned tenants with a matching policy file
  are authorized; requests/responses are bounded and timed out, concurrent
  clients are capped, and shutdown drains active workers safely
- Private-home and read-only dataset mounts
- Start, stop, status, shell, execute, and remove operations
- Extension and `mt-container` management
- Remote-SSH terminal access
- Dev Containers attachment to a running workspace
- Dev Containers metadata that defaults file dialogs to `/workspace`
- Network-disabled or egress policy
- Workspace metadata and account-cleanup integration
- Configurable CPU, memory, and PID limits
- Live per-workspace CPU/RAM/PID accounting from rootless Podman/Docker stats,
  with authenticated-tenant aggregates in the GPU Panel and
  `mt-container stats <id>`
- Container process inspection in the GPU Panel and `mt-container ps`
- Recent container logs in the GPU Panel and `mt-container logs`
- Validated per-process signaling from the GPU Panel and `mt-container kill`
- Detached non-interactive workspace jobs in `mt-container`
  (`job-run`/`jobs`/`job-status`/`job-logs`/`job-cancel`) with per-job status,
  logs, cancellation, optional wait-for-queued-workspace admission, optional
  wall-clock time limits, and retry policy stored/enforced from the tenant
  home; `job-prune` removes old terminal job
  records on demand, and the server's cleanup loop prunes old terminal records
  automatically via `JOB_RETENTION_DAYS`
- Job-first workspace submission bridge: `mt-container job-submit` creates or
  queues a new workspace request with GPU count/priority, waits for GPU
  admission, then starts a detached job in that workspace
- Persistent per-tenant named volumes with account-lifecycle cleanup
- Administrator-gated custom OCI image pull and workspace creation
- Tenant-home image/Dockerfile `devcontainer.json` parsing, build, and rebuild
- Constrained Compose `devcontainer.json` workflows: one primary service,
  tenant-home Compose files, generated MTGPU override for labels/mounts/GPU
  devices/network/ports, and Compose-backed start/stop/remove
- Per-GPU selection and exclusive reservation across container workspaces
  (persisted in `.mtgpu/gpu-reservations.json`; maximum is the detected GPU
  count)
- Linux/NVIDIA container-admission coordination with native GPU processes:
  currently occupied native devices are excluded, and an undersized multi-GPU
  request waits in the FIFO queue instead of silently assigning fewer GPUs
- Linux best-effort native shell guard for container-reserved GPUs:
  provisioning refreshes the managed shell block in `.profile`/`.bashrc` so
  ordinary SSH shells run `mt-container native-env` and set
  `CUDA_VISIBLE_DEVICES` away from GPUs reserved by managed containers unless
  the user already set it; the same command exports native-busy and
  native/container conflict GPU lists for diagnostics
- Automated dangling-image pruning and GPU-reservation/orphan reconciliation
  (periodic background pass plus account-lifecycle cleanup); reservation state
  is atomically persisted mode 0600, reconstructed from exact live workspace
  assignments after file loss, and preserved when a tenant registry/runtime
  cannot be safely inspected
- Registry credentials for private images: server-side admin pre-pull with a
  confidential `registryAuthfile` shared to tenants via exported tarballs
  (`prePullImages`/`sharedImageDir`), plus per-tenant `podman login` for a
  tenant's own private pulls
- Runtime secret-file injection (`CONTAINERS.runtimeSecrets`): admin-listed
  secret files are staged per tenant (owned by the tenant, mode 0400) and
  bind-mounted read-only at a target path (default `/run/secrets/<name>`) for
  matching `accounts`/`templates`; removed with the workspace; status reports
  mounted secret names/targets (never values)
- Build-time secret injection for devcontainer Dockerfile builds: the same
  `runtimeSecrets` are passed to `podman build --secret id=<name>,src=...` so a
  Dockerfile can `RUN --mount=type=secret,id=<name>` without baking values into
  image layers (podman native; Docker requires BuildKit)
- Published ports with an egress policy (`networkPolicy`, workspace
  `publishPorts`, `CONTAINERS.publishHost`): workspaces publish only the ports
  selected by the user; ports bind on
  loopback by default, or on `0.0.0.0` only when the administrator explicitly
  opts in, with auto-assigned host ports reported in status for tunneling/direct
  links; the server also writes an admin-readable
  `.mtgpu/container-port-routes.json` manifest with current running workspace
  routes for a future proxy/router
- Client port auto-forwarding: the extension forwards each published host port to
  the client's localhost with a managed companion SSH connection (`ssh -L`) and
  shows clickable local links in the GPU Panel; forwards are torn down on
  stop/remove/switch
- Multi-vendor live GPU detection and utilization sampling through the reused
  `remoteGPU-cpp-server/gpu_utils.cpp` layer: NVIDIA uses `nvidia-smi`; AMD
  attempts ROCm SMI and falls back to `radeontop`; Intel attempts
  `intel_gpu_top` JSON and falls back to text busy percentages. Missing tools or
  unsupported formats degrade to zero samples rather than disabling the server.
- Historical resource usage: a background sampler records per-GPU
  utilization/memory into a bounded, persisted ring buffer
  (`.mtgpu/usage-history.json`); the GPU Panel shows recent per-GPU sparklines
- Historical per-tenant container CPU/RAM/PID attribution in usage history: the
  sampler records authenticated-tenant running-container aggregates from
  rootless Podman/Docker stats and the GPU Panel shows a **Containers**
  sparkline
- Linux/NVIDIA per-tenant native/container GPU-memory and process-count
  accounting plus estimated GPU-utilization attribution in usage history, using
  `nvidia-smi pmon` per-process SM samples when available and a process-count
  fallback otherwise; client responses expose only the authenticated tenant's
  attribution
- Vendor-neutral container GPU device injection through CDI selectors generated
  from detected GPU vendors: `nvidia.com/gpu=<entry>`, `amd.com/gpu=<entry>`,
  and `intel.com/gpu=<entry>`. CPU-only containers no longer require NVIDIA
  tooling. Vendor CDI specs and matching in-container user-space libraries remain
  host/image responsibilities.
- One-click official-VS-Code handoff from the local GPU Panel through
  Remote-SSH into the exact selected running workspace; the handoff is
  short-lived, one-use, and targets the full runtime container ID

### Still missing

- Routed port exposure with reverse proxy, hostnames, authentication, and TLS
  termination (direct public host binding and an admin route manifest are
  implemented, but MTGPU does not yet run an authenticated HTTPS proxy/router)
- Hard bidirectional scheduling enforcement with native GPU processes: container
  admission avoids devices already busy at creation time, and Linux shells get a
  best-effort `CUDA_VISIBLE_DEVICES` guard plus conflict diagnostics. Missing:
  OS/kernel/device-cgroup enforcement that prevents a deliberate native SSH
  process from opening a GPU already reserved by a managed container, and
  similarly guarantees that containers cannot take GPUs assigned to native jobs.
- Advanced batch scheduler with separate GPU admission: queued workspaces support
  priority ordering, detached jobs can wait for queued workspace admission, and
  `job-submit` can create/queue a workspace then start a detached job. Missing:
  a durable server-side batch scheduler where users submit the job first and the
  server independently owns the full lifecycle: queue, GPU admission,
  workspace/image selection, command execution, retry/timeout, logs, and
  post-job workspace retention/removal policy even if the CLI exits.
- Portable exact per-process GPU-utilization attribution across all vendors and
  hosts: Linux/NVIDIA uses `nvidia-smi pmon` per-process SM samples when
  available and falls back to splitting the per-GPU sample across active GPU
  processes. Missing: exact and portable per-process utilization for drivers and
  platforms that do not expose equivalent metrics, including non-NVIDIA GPUs,
  Windows/WSL host differences, MIG/MPS/shared contexts, and other vendor APIs.
- Full cross-platform and cross-vendor container validation: Linux/rootless
  Podman or Docker is the first-class container host. NVIDIA, AMD, and Intel CDI
  selectors are generated, but non-NVIDIA GPU container stacks still need
  dedicated validation matrices for toolkit install, CDI spec generation,
  matching image user-space libraries, group/device permissions, WSL behavior,
  process/log/port inspection, cleanup, reconciliation, and security behavior.

## Container architecture and roadmap

Container support is now an optional, integrated workspace backend that
complements, rather than replaces, the native SSH and Python workflow. Both
modes operate side-by-side on the same server:

- **Native workspaces** continue to use the provisioned OS account, Remote-SSH,
  `~/.venv`, shared interpreters, and shared datasets directly on the host.
- **Container workspaces** add reproducible, project-specific environments while
  reusing the same DirectLink identity, OS account, private home, datasets,
  quotas, GPU visibility, and retention policy.

Users who do not need containers should see no additional setup or UI. Container
software must not become a prerequisite for server startup, authentication,
account provisioning, Remote-SSH, terminal access, or Python selection. The
server instantiates the container backend only when it is enabled in
configuration and its host readiness checks pass. Missing Podman/Docker, GPU
toolkit/CDI support, or platform support produces an unavailable capability,
not a provisioning failure.

### Current architecture

The active implementation lives in this repository:

- `src/ContainerService.*` owns tenant workspace registries, GPU reservations,
  queue promotion, lifecycle operations, runtime/build secrets, port-route
  manifests, statistics, reconciliation, and account cleanup.
- `commands/mt-container.cpp` is a thin tenant CLI. It talks to the running
  server over the local Unix control socket and is authenticated with
  `SO_PEERCRED`, so the CLI and GPU Panel share the same server-side state.
- `src/LocalControlServer.h` is the local broker used by `mt-container`.
- `src/ContainerProtocol.h` defines the additive DirectLink messages used by
  the extension/client module to query capabilities and manage workspaces.
- `src/ContainerConfig.h` and `src/ServerConfig.*` parse the `CONTAINERS`
  policy.

The earlier standalone container execution spike supplied the initial
`ContainerEngine`, `ContainerTemplates`, and `Subprocess` designs. The active
implementations and all host-setup scripts are now owned by this repository.
The production MTGPU path is the persistent-workspace service described here.

### Workspace behavior

Managed container workspaces are persistent workspaces, not only ephemeral batch
jobs:

1. Create a workspace from a curated OCI image or a repository containing
   `devcontainer.json`.
2. Bind the selected project or private home read-write and configured shared
   datasets read-only. Existing host storage remains the source of truth and
   survives container replacement.
3. Select an allowed GPU and apply CPU, memory, process, disk, and execution
   limits shared with native workloads.
4. Start, stop, inspect, rebuild, and remove the workspace independently of the
   user's OS account.
5. Open the project through the established Remote-SSH and Dev Containers
   workflow, including one-click official-VS-Code handoff to the exact selected
   running container.
6. Report container state, image/template, processes, logs, ports, and resource
   usage in the GPU panel.

### Isolation and lifecycle requirements

- Run containers through a constrained server-side broker; never expose the
  Docker socket or grant tenant accounts unrestricted engine administration.
- Associate every workspace with its authenticated tenant and execute it under
  an appropriate per-user rootless/container identity. Validate every bind
  mount against that tenant's home and configured shared directories.
- Keep networking disabled by default and make egress and published ports
  explicit policy decisions. Add private-registry credentials and secrets
  without placing them in public templates or logs.
- Allocate specific GPUs instead of exposing all devices, and coordinate GPU
  scheduling between native processes and containers.
- Persist workspace metadata and reconcile it with the runtime after server
  restart. Clean up orphaned containers, stale images, and expired workspaces
  according to account retention policy.
- Treat Linux as the first-class container host. Windows may use WSL2 after
  validation; macOS remains useful as a client/native host but does not provide
  the standard NVIDIA CUDA container model.

### Implementation status snapshot

- `CONTAINERS.enabled` is a runtime feature switch and defaults to `false`.
- `MTGPU_ENABLE_CONTAINER_SUPPORT` controls whether the Linux container adapter
  is compiled. A native-only build continues to provide every SSH and Python
  feature.
- The adapter uses the engine, template, and subprocess components maintained
  directly in this repository.
- Detection runs under the authenticated tenant's OS identity, even though the
  account-management server itself is elevated. It therefore inspects the
  tenant's rootless runtime rather than a root-owned image store.
- `MSG_TYPE_QUERY_CONTAINER_CAPABILITIES` reports disabled, unavailable, or
  ready status; engine/rootless/GPU-toolkit details; policy flags; and the
  configured template catalogue.
- The client module exposes the additive query. The extension hides container
  UI when the feature is disabled, shows a readiness reason when an
  administrator enabled it but the tenant runtime is unavailable, and shows
  engine/template details when ready. Older servers keep the existing panel
  unchanged.

Example configuration:

```json
"CONTAINERS": {
  "enabled": false,
  "allowCustomImages": false,
  "networkPolicy": "disabled",
  "publishHost": "127.0.0.1",
  "pidsLimit": 1024,
  "volumes": [
    {
      "name": "package-cache",
      "mountPath": "/mtgpu/cache"
    }
  ],
  "templates": [
    {
      "id": "pytorch-cu126",
      "name": "PyTorch - CUDA 12.6",
      "image": "registry.example.com/mtcode/pytorch-cu126:1"
    }
  ]
}
```

Install or repair the Linux container host prerequisites with:

```sh
./install_container.sh
```

The script checks Podman/Docker, rootless UID helpers, subuid/subgid allocation,
and cgroup v2 without changing the host. On NVIDIA hosts it also checks the
NVIDIA driver, NVIDIA Container Toolkit, and CDI device configuration. On
non-NVIDIA or CPU-only hosts it invokes the lower-level installer with
`--skip-nvidia`, so rootless containers can be provisioned without forcing an
NVIDIA stack. If anything is missing, it lists the required work and asks for
approval before invoking the privileged installer. Use
`--engine podman|docker`, `--disable-egress`, or `--yes` for custom or
unattended host provisioning. Egress support is enabled by default. A complete
NVIDIA GPU-in-container smoke test runs only on NVIDIA hosts; AMD/Intel smoke
tests are roadmap items tied to vendor-specific sample images.

An empty `templates` array publishes no templates. `networkPolicy`
accepts `disabled` or `egress` and is enforced when the workspace is created.

The reused subprocess layer resolves an OS account and drops supplementary
groups, gid, and uid before invoking the container CLI. Tenant containers are
created and managed through this identity-aware path; invoking tenant containers
through the server's root identity is intentionally unsupported.

### Implemented persistent workspace lifecycle

The implemented lifecycle provides multiple persistent workspaces per tenant:

- Each workspace is created inside the authenticated tenant's rootless
  Podman/Docker context and has a collision-free generated name containing the
  hex-encoded account and per-tenant workspace ID.
- The user selects from the server's allowed template catalogue or, when the
  administrator enables it, supplies a validated custom OCI image reference.
  The image is pulled into that tenant's rootless image store when first needed.
- The tenant home is mounted read-write at `/workspace`; each configured
  dataset is mounted read-only at its original absolute host path. This keeps
  the `~/shared/<name>` symlinks provisioned for native SSH valid inside the
  container as `/workspace/shared/<name>`.
- The container receives `no-new-privileges`, the configured PID safety limit
  and network policy, and vendor CDI device selectors when GPUs are
  requested. The engine socket is never mounted into the container.
- Create allocates a new ID up to `maxWorkspacesPerTenant`. Query, start, stop,
  and remove target a selected ID through `mtgpu-client-module`, the Unix-socket
  CLI broker, and the GPU panel's **Container Workspace** action.
- Removing a container does not remove the tenant home, shared datasets, or
  configured persistent named volumes. Workspace registries are stored at
  `<ROOT_FOLDER>/.mtgpu/workspaces/<account>.json`.
- Inactivity cleanup removes a recorded workspace before deleting its OS
  account and home. Account deletion is deferred if container cleanup fails,
  preventing an orphaned tenant process.
- When containers are enabled, Linux account provisioning best-effort enables
  systemd linger and prepares `/run/user/<uid>` for that tenant. Missing
  subuid/subgid ranges are reported without breaking native SSH provisioning.

The workspace menu provides two developer entry paths while it is running:

- **Open Terminal** opens an interactive `/bin/bash` through `podman exec` or
  `docker exec`. From a local window the command travels through the existing
  mapped SSH endpoint; from a Remote-SSH window it runs directly on the host.
- **Attach with Dev Containers** passes the full running-container ID to Dev
  Containers. From a local official VS Code window, a short-lived one-use state
  transfer opens Remote-SSH and resumes the exact attach automatically. For
  Podman hosts, MTGPU sets `dev.containers.dockerPath` to `mt-podman` so Dev
  Containers sees the MTGPU-managed container store.

### SSH container command

When container support is enabled, account provisioning installs
`~/.local/bin/mt-container` and writes the server's container policy
(including the server's control-socket path) to `~/.mtgpu/container-policy.json`.
The CLI is a **thin client of the running server**: it connects to the local
Unix control socket (the server authenticates the caller by uid via
`SO_PEERCRED`) and delegates create/list/start/stop/remove/signal to the same
`ContainerService` the GPU Panel uses. The CLI therefore shares the **same
per-tenant workspace registry, GPU reservations, and FIFO queue** as the panel —
the two interfaces always see the same set of workspaces. Only the interactive
`shell`/`exec` run `podman exec` directly (they need a local TTY), on the
container name the server reports.

```bash
mt-container templates
mt-container list                       # all workspaces: ID, STATE, TEMPLATE, GPUs, queue#
mt-container create pytorch-2.4-cuda12.4 --gpu-count 1 --priority 10
mt-container create pytorch-2.4-cuda12.4 --gpu-indexes 0,1 --priority 10
mt-container create --image ghcr.io/example/team-image:latest --priority 10
mt-container create --devcontainer .devcontainer/devcontainer.json --priority 10
mt-container start <id>                 # id optional when only one workspace exists
mt-container stop <id>
mt-container rebuild <id>
mt-container ps <id>
mt-container logs <id>
mt-container stats <id>
mt-container kill <id> <pid> [TERM|KILL|INT|HUP]
mt-container native-env                 # shell exports that hide container-reserved GPUs
mt-container shell <id>
mt-container exec <id> -- python /workspace/Projects/example/train.py
mt-container job-run <id> [--timeout SECONDS] [--retries N] -- python /workspace/Projects/example/train.py
mt-container jobs
mt-container job-status <job-id>
mt-container job-logs <job-id>
mt-container job-cancel <job-id>
mt-container job-prune [--older-than DAYS] [--dry-run]
mt-container remove <id>
```

`create` takes a template id or, when `allowCustomImages` is enabled, `--image`
or `--devcontainer`, plus optional `--gpu-count N`
(1..detected GPU count; omitted means no GPU visibility),
`--gpu-indexes 0,1` for specific GPU indexes, and `--priority N` (-100..100;
default 0). The old `--gpus N` spelling is accepted as an alias for
`--gpu-count N`.
If the GPUs are busy the workspace is
**queued** and the server's scheduler starts it automatically. A workspace id
selects the target for the per-workspace commands (omit it when only one
exists). Removing a workspace removes only its container;
tenant files and shared datasets remain. Reconnect once after a server upgrade
to refresh the installed command and policy manifest.

Remaining intentional limitations: Compose devcontainers are limited to a
single selected primary service with MTGPU-managed overrides; published ports
bind to loopback by default and can bind directly to `0.0.0.0` only when the
administrator explicitly opts in, but MTGPU does not yet provide an
authenticated HTTPS reverse proxy/router; a later native SSH process is not yet
kernel-prevented from selecting a container-reserved GPU. Image preparation
reports a single progress spinner rather than streamed pull progress.

## Caveats / TODO

- Disk quotas depend on the filesystem (ext4 `usrquota` / XFS prjquota /
  Windows volume quotas). When unavailable, MTGPU can still report measured
  usage and warn, but it cannot provide hard filesystem enforcement on that
  mount. See "Disk quota enablement and enforcement" for the installer/startup
  split.
- The unused `MAX_EXECUTION_MINUTES_DAY` and `MAX_CONCURRENT_EXECUTION`
  settings were removed. Workspace and job admission use their actual
  GPU/CPU/RAM requirements; no unenforced execution-limit policy is displayed
  as active configuration.
- Windows/macOS account paths are best-effort and need testing on those OSes.

## Files

| File | Purpose |
|------|---------|
| `multi-interpreters-design.md` | multi-interpreter schema, layout, protocol, and maintenance contract |
| `install_container.sh` | approval-gated Linux/WSL2 container host readiness and installation |
| `commands/mt-python.cpp` | standalone SSH-terminal interpreter selector |
| `commands/mt-container.cpp` | standalone SSH-terminal managed-container command |
| `src/ServerConfig.*` | `config.json` load/save/validate |
| `src/UserAccountManager*.*` | cross-platform OS account / quota / `.ssh` |
| `src/SshKeyManager.*` | `REGISTER_SSH_KEY` → `authorized_keys` |
| `src/SshServerManager.*` | dedicated `sshd` lifecycle + config/host keys |
| `src/MtServerCliLauncher.*` | launches `mtserver-cli` (appId `GPU_SERVER_SSH`) |
| `src/UserRegistry.*` | per-tenant activity / keep state (`.mtgpu/users.json`) |
| `src/CleanupManager.*` | `TENANT_RETENTION_DAYS` sweep |
| `src/GpuServerCore.*` | provisioning + message dispatch |
| `src/main.cpp` | startup orchestration + connection loop |
