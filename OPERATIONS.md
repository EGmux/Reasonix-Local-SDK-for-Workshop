# deepseek-reasonix — Operations Procedures

This document covers the full lifecycle of the **deepseek-reasonix** SDK: from
prerequisites and local iteration through building, publishing, consuming in a
workshop, and troubleshooting.

> **Reference material**
>
> - [workshop CLI](https://ubuntu.com/workshop/docs/explanation/workshops/workshop-cli/)
> - [sdkcraft CLI](https://ubuntu.com/workshop/docs/reference/cli/sdkcraft/)
> - [workshopctl CLI](https://ubuntu.com/workshop/docs/reference/cli/workshopctl/)
> - [How to build an SDK](https://ubuntu.com/workshop/docs/how-to/develop-sdks/build-an-sdk/)
> - [How to publish an SDK](https://ubuntu.com/workshop/docs/how-to/develop-sdks/publish-an-sdk/)

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Project Structure](#2-project-structure)
3. [Local Build and Iteration](#3-local-build-and-iteration)
4. [Trying the SDK in a Workshop](#4-trying-the-sdk-in-a-workshop)
5. [Running SDK Tests](#5-running-sdk-tests)
6. [Publishing to the SDK Store](#6-publishing-to-the-sdk-store)
7. [Consuming the Published SDK](#7-consuming-the-published-sdk)
8. [Launching and Using the Workshop](#8-launching-and-using-the-workshop)
9. [Refreshing the Workshop](#9-refreshing-the-workshop)
10. [Health Checks and Debugging](#10-health-checks-and-debugging)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Prerequisites

| Component | Minimum version | Install command |
|---|---|---|
| **SDKcraft** | latest | `snap install sdkcraft` |
| **Workshop** | latest | `snap install workshop` |
| LXD | 6.6+ | `snap install lxd && lxd init` |
| Ubuntu One account | — | Sign up at [login.ubuntu.com](https://login.ubuntu.com) |

Verify all three are installed and reachable:

```shell
sdkcraft version
workshop version
lxc version
```

> **Note:** SDKcraft is a separate snap from Workshop. Both must be installed.
> SDKcraft is used by SDK *publishers*; Workshop is used by SDK *consumers*.
> As an SDK publisher you will use both.

---

## 2. Project Structure

The SDK source tree lives at `deepseek-reasonix/` and contains:

```
deepseek-reasonix/
├── sdkcraft.yaml           # SDK build definition (metadata, parts, plugs, slots)
├── VERSION                 # Upstream version pin ("1.0.0")
├── workshop.yaml           # Workshop definition for local try/iteration
├── OPERATIONS.md           # This document
└── hooks/
    ├── setup-base         # System-level setup (runs as root)
    ├── setup-project      # User-level setup  (runs as workshop user)
    └── check-health       # Health verification (runs as root)
```

### 2.1 Definition files at a glance

| File | Purpose |
|---|---|
| `sdkcraft.yaml` | Consumed by **sdkcraft** at build time. Declares platforms, parts, plugs, and slots. |
| `workshop.yaml` | Consumed by **workshop** at launch time. References the SDK and wires connections. |
| `VERSION` | Read by **SDKcraft** at build time via `$CRAFT_PROJECT_VERSION`. Keep it quoted. |

### 2.2 Hook execution order

At `workshop launch` or `workshop refresh`, hooks run in this sequence:

```
setup-base     →   setup-project     →   check-health
  (root)              (workshop user)      (root)
```

- **setup-base** — Installs Node.js 22 LTS from NodeSource, installs `nodejs`,
  `build-essential`, and `git`, and exports `DEEPSEEK_CACHE_DIR` globally via
  `/etc/profile.d/deepseek-reasonix.sh`.
- **setup-project** — Runs `sudo npm install -g reasonix dsnix` and creates
  `/home/workshop/.cache/reasonix`.
- **check-health** — Executes `dsnix --help` as the `workshop` user and reports
  the result via `workshopctl set-health okay` or `workshopctl set-health error`.

### 2.3 Interfaces declared by the SDK

| Name | Kind | Interface | Purpose |
|---|---|---|---|
| `agent-cache` | plug | `mount` | Persists agent cache at `/home/workshop/.cache/reasonix` across refreshes |
| `api-connectivity` | slot | `tunnel` | Declares TCP/443 connectivity for outbound API calls |

The mount plug auto-connects to `system:mount` at launch. The tunnel slot is
wired explicitly through the `connections` section in `workshop.yaml`.

---

## 3. Local Build and Iteration

### 3.1 Pack the SDK

From the `deepseek-reasonix/` directory:

```shell
cd deepseek-reasonix
sdkcraft pack
```

This produces one `.sdk` artifact per platform declared in `sdkcraft.yaml`:

```
deepseek-reasonix_amd64_ubuntu@24.04.sdk
deepseek-reasonix_amd64_ubuntu@26.04.sdk
```

### 3.2 Incremental rebuild

After editing hooks or the YAML definition:

```shell
sdkcraft clean && sdkcraft pack
```

Omit `clean` for small changes that **SDKcraft** can incrementally rebuild.

### 3.3 Build a single platform

```shell
sdkcraft pack --platform ubuntu@24.04:amd64
```

---

## 4. Trying the SDK in a Workshop

### 4.1 Copy the SDK into the try area

```shell
sdkcraft try
```

This packs the SDK and copies the artifacts into the **Workshop** try area
(`$XDG_DATA_HOME/workshop/try/`). The SDK becomes available to workshops that
reference it with the `try-` prefix.

### 4.2 Create a workshop definition

A `workshop.yaml` is already provided in the project root:

```yaml
name: reasonix-dev
base: ubuntu@24.04
sdks:
  - name: try-deepseek-reasonix
  - name: system
    plugs:
      deepseek-api:
        interface: tunnel
        endpoint: 443/tcp
connections:
  - plug: system:deepseek-api
    slot: deepseek-reasonix:api-connectivity
actions:
  ask: |
    dsnix ask "$@"
  code: |
    dsnix code "$@"
  help: |
    dsnix --help
```

> **Note:** `try-deepseek-reasonix` is used during local iteration.
> Replace with `deepseek-reasonix` (no prefix) once the SDK is published to the Store.

### 4.3 Launch the workshop

```shell
workshop launch --verbose --wait-on-error
```

| Flag | Purpose |
|---|---|
| `--verbose` | Prints hook output to the terminal in real time |
| `--wait-on-error` | Leaves the workshop running on failure so you can `workshop shell` into it |

### 4.4 Verify the workshop status

```shell
workshop info reasonix-dev
```

Look for:

```
status:   ready
sdks:
  deepseek-reasonix:
    status:  okay
    ...
```

### 4.5 Exercise the SDK

```shell
# Run a named action
workshop run reasonix-dev -- help

# Open an interactive shell
workshop shell reasonix-dev

# Run an ad-hoc command
workshop exec reasonix-dev -- dsnix ask "What is the meaning of life?"
```

### 4.6 Stop and remove the workshop

```shell
workshop stop reasonix-dev
workshop remove reasonix-dev
```

---

## 5. Running SDK Tests

If the SDK ships a `tests/` directory with
[spread](https://github.com/canonical/spread) tests:

```shell
sdkcraft test
```

**SDKcraft** provisions a clean LXD container for each test, installs the
packed SDK, and runs the declared scenarios.

List the test jobs that would run:

```shell
sdkcraft test --list
```

Run a specific suite:

```shell
sdkcraft test main/smoke
```

---

## 6. Publishing to the SDK Store

### 6.1 Authenticate

```shell
sdkcraft login
sdkcraft whoami           # confirm the active account
```

### 6.2 Register the SDK name (one-time)

```shell
sdkcraft register deepseek-reasonix
```

This reserves the name globally on the SDK Store. It cannot be changed
after the first release.

### 6.3 Upload artifacts

Upload each `.sdk` file produced by `sdkcraft pack`:

```shell
sdkcraft upload deepseek-reasonix_amd64_ubuntu@24.04.sdk
sdkcraft upload deepseek-reasonix_amd64_ubuntu@26.04.sdk
```

Each upload returns a revision number (e.g., `1`, `2`).

### 6.4 Upload and release in one step

```shell
sdkcraft upload deepseek-reasonix_amd64_ubuntu@24.04.sdk --release latest/edge
```

### 6.5 Release a revision to channels

```shell
sdkcraft release deepseek-reasonix 1 latest/edge
sdkcraft release deepseek-reasonix 1 latest/beta,latest/stable
```

Channels follow the pattern `[<TRACK>/]<RISK>[/<BRANCH>]`:

| Part | Values |
|---|---|
| track | `latest` (default), or a version like `1.x` |
| risk | `edge`, `beta`, `candidate`, `stable` |
| branch | Optional, auto-expires after 30 days |

### 6.6 Create additional tracks (if needed)

```shell
sdkcraft create-track deepseek-reasonix --track 1.x
```

### 6.7 Automate with CI

The canonical
[template-sdk](https://github.com/canonical/template-sdk) ships GitHub
Actions workflows that:

1. Build on pull requests.
2. Run `sdkcraft upload --release` on push to the version branch.

Configure the following secrets in your repository:

| Secret | Value |
|---|---|
| `SDKCRAFT_STORE_CREDENTIALS` | Exported credentials from `sdkcraft login` |

---

## 7. Consuming the Published SDK

Once a revision is released to a channel, any **Workshop** user can consume it:

```yaml
# workshop.yaml
name: reasonix-dev
base: ubuntu@24.04
sdks:
  - name: deepseek-reasonix
    channel: latest/stable
  - name: system
    plugs:
      deepseek-api:
        interface: tunnel
        endpoint: 443/tcp
connections:
  - plug: system:deepseek-api
    slot: deepseek-reasonix:api-connectivity
actions:
  ask:  dsnix ask "$@"
  code: dsnix code "$@"
  help: dsnix --help
```

| Field | Value | Notes |
|---|---|---|
| `base` | `ubuntu@24.04` or `ubuntu@26.04` | Must match one of the SDK's platforms |
| `channel` | `latest/stable` | Any risk level the publisher has released to |

---

## 8. Launching and Using the Workshop

### 8.1 Launch

```shell
workshop launch --verbose --wait-on-error
```

On success the workshop transitions to `ready` status.

### 8.2 View workshop details

```shell
workshop info reasonix-dev
workshop list                       # all workshops in the project
workshop connections --all          # interface wiring
```

### 8.3 Run named actions

```shell
workshop run reasonix-dev -- ask "How do I refactor this Python class?"
workshop run reasonix-dev -- code "Write a FastAPI endpoint for user login"
workshop run reasonix-dev -- help
```

Actions are defined in the `actions:` section of `workshop.yaml`. They are
injected at run time and forward `"$@"` positional arguments.

### 8.4 Interactive shell

```shell
workshop shell reasonix-dev
```

Inside the shell the `DEEPSEEK_CACHE_DIR` environment variable is available,
and `dsnix` is on `PATH`.

### 8.5 Ad-hoc commands

```shell
workshop exec reasonix-dev -- dsnix --version
workshop exec reasonix-dev -- echo $DEEPSEEK_CACHE_DIR
```

---

## 9. Refreshing the Workshop

Apply changes to `workshop.yaml` without rebuilding from scratch:

```shell
workshop refresh reasonix-dev --verbose --wait-on-error
```

**Workshop** takes ZFS snapshots after each SDK's `setup-base` hook. At refresh
time, unchanged SDKs are restored from their snapshots; only added, removed, or
changed SDKs go through a fresh install.

| Operation | Effect on connections |
|---|---|
| Refresh | Re-evaluates auto-connect; drops manual `workshop connect` connections |
| Restore | Rolls back to the last successful launch/refresh snapshot; drops manual connections |

To restore a workshop to its last known-good state:

```shell
workshop restore reasonix-dev --verbose
```

---

## 10. Health Checks and Debugging

### 10.1 Check health

```shell
workshop info reasonix-dev
```

The SDK's `status` field reports one of:

| Status | Meaning |
|---|---|
| `okay` | The SDK is functional |
| `waiting` | `check-health` should be retried (up to 10×, 1 s apart) |
| `error` | A hook failed or `set-health error` was called |

### 10.2 Review task history

```shell
workshop changes reasonix-dev
workshop tasks reasonix-dev          # detailed task list
workshop tasks reasonix-dev --last   # most recent failed task
```

### 10.3 Debug a failed hook

If a hook fails, launch with `--wait-on-error` to keep the workshop in
`waiting` state, then inspect it:

```shell
workshop launch --verbose --wait-on-error
workshop shell reasonix-dev          # inspect the environment
# Inside the shell:
cat /var/lib/workshop/sdk/deepseek-reasonix/sdk/hooks/setup-base
/var/lib/workshop/sdk/deepseek-reasonix/sdk/hooks/setup-base   # re-run manually
```

### 10.4 Set health manually (inside the workshop)

```shell
workshopctl set-health okay
workshopctl set-health error "dsnix binary not found"
workshopctl set-health --code=no-node error "Node.js is not installed"
```

The `check-health` hook in this SDK uses this mechanism.

### 10.5 View workshop logs

```shell
workshop changes reasonix-dev
workshop tasks reasonix-dev
```

---

## 11. Troubleshooting

### 11.1 Hook fails with "command not found"

The `setup-base` hook installs Node.js from NodeSource. If `curl` or `apt-get`
fail, check network connectivity inside the workshop:

```shell
workshop shell reasonix-dev
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
```

### 11.2 npm install fails in setup-project

The `setup-project` hook runs as the `workshop` user and uses `sudo` for the
global npm install. If sudo is not configured for passwordless execution, add:

```shell
echo "workshop ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/workshop
```

### 11.3 Mount plug not connected

Mount plugs auto-connect by default. Verify with:

```shell
workshop connections --all
```

Look for:

```
mount  deepseek-reasonix:agent-cache  system:mount  auto
```

If missing, connect manually:

```shell
workshop connect reasonix-dev/deepseek-reasonix:agent-cache system:mount
```

### 11.4 Tunnel slot not connected

Tunnel interfaces are not auto-connected. The `workshop.yaml` must include an
explicit `connections` entry:

```yaml
connections:
  - plug: system:deepseek-api
    slot: deepseek-reasonix:api-connectivity
```

If the connection is missing, add it to the definition and refresh:

```shell
workshop refresh reasonix-dev
```

### 11.5 "error" status after launch

1. Run `workshop tasks reasonix-dev --last` to find the failing hook.
2. Re-launch with `--verbose --wait-on-error` for real-time output.
3. Shell into the workshop and re-run the hook manually.

### 11.6 SDK doesn't appear in workshop list

Ensure `sdkcraft try` was run before `workshop launch` when using the
`try-` prefix. Verify the try area:

```shell
ls $XDG_DATA_HOME/workshop/try/
```

### 11.7 Version mismatch

If `sdkcraft pack` fails or produces unexpected artifacts, verify the
`version` field in `sdkcraft.yaml` is quoted:

```yaml
version: "1.0.0"    # ✓ quoted — always use quotes
```

---

## Quick Reference Card

```shell
# === SDK PUBLISHER ===

# Build
sdkcraft pack                          # Build all platforms
sdkcraft pack --platform ubuntu@24.04:amd64   # Single platform

# Try locally
sdkcraft try                           # Pack + copy to try area
workshop launch --verbose --wait-on-error     # Launch with try SDK
workshop shell reasonix-dev                   # Inspect
workshop remove reasonix-dev                  # Clean up

# Test
sdkcraft test                          # Run spread tests
sdkcraft test --list                   # List test jobs

# Publish
sdkcraft login                         # Authenticate
sdkcraft register deepseek-reasonix    # Register name (once)
sdkcraft upload <FILE>.sdk             # Upload revision
sdkcraft release deepseek-reasonix 1 latest/stable   # Release

# === WORKSHOP USER ===

workshop launch --verbose              # Build workshop
workshop info reasonix-dev             # Check status
workshop run reasonix-dev -- ask "..." # Run action
workshop shell reasonix-dev            # Interactive shell
workshop exec reasonix-dev -- dsnix --help  # Ad-hoc command
workshop refresh reasonix-dev          # Apply definition changes
workshop remove reasonix-dev           # Tear down
```

---

# CRITICAL 

When remount local sdk's make sure to do with the following syntax
`workshop remount $WORKSHOP/$SDK:$PLUG $DIR`
and not with
`workshop remount $WORKSHOP/try-$SDK:$PLUG $DIR`

*Document version 1.0 — Corresponds to deepseek-reasonix SDK v1.0.0*
*Refer to the [Workshop documentation](https://ubuntu.com/workshop/docs/) for
the most up-to-date CLI reference and conceptual guides.*
