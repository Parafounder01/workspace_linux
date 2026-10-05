# Dev Workspace: Windows 2022 + Tailscale RDP

Temporary cloud desktop on a GitHub Actions runner, reachable only over Tailscale.
**Software-only.** No USB/serial/JTAG access.

## Which environment for which work

| Work | Use |
|---|---|
| Embedded software | Local PC with PlatformIO or VS Code, plus simulators (Wokwi, Renode, QEMU) when no board is available |
| Hardware (flash, debug, logic analyzer) | Lab PC with the board attached, reached via Tailscale RDP/SSH. Persistent, real USB access |
| Data analysis | Jupyter locally, Colab, or GitHub Codespaces |
| Longer cloud desktop (>6 h, persistent) | Small Azure/AWS Windows VM, not Actions |

This workflow is for the gaps: a throwaway Windows box for firmware builds, simulation, and analysis when your local PC is unavailable.

## Setup

Repo -> Settings -> Secrets and variables -> Actions:

| Secret | Value |
|---|---|
| `RDP_PASSWORD` | Strong password for local user `RDP` |
| `TAILSCALE_AUTHKEY` | **Ephemeral**, **tagged** (`tag:gh-runner`), preferably single-use key |

Tailscale ACL (admin console) - allow only your devices to reach the tag:

```json
{
  "tagOwners": { "tag:gh-runner": ["autogroup:admin"] },
  "acls": [
    { "action": "accept", "src": ["autogroup:member"], "dst": ["tag:gh-runner:3389"] }
  ]
}
```

## Run

1. Actions -> **Dev workspace** -> Run workflow (set `session_minutes`, max 335).
2. Wait for the "Keep session alive" step. Log shows Tailscale IP and username.
3. Join the same tailnet locally, then RDP to `<tailscale-ip>`, user `RDP`, password = `RDP_PASSWORD`.
4. Work in `C:\work`.
5. End early: create `C:\work\STOP`.
6. Download the `workspace-<run_id>` artifact (kept 7 days).

## Share files and folders

Three ways, all Tailscale-only:

**1. SMB share** (`C:\work` -> `\\<tailscale-ip>\work`, user `RDP`, password = `RDP_PASSWORD`). Best for folders and large transfers.

- Windows: `net use W: \\<tailscale-ip>\work /user:RDP *`
- Linux: `sudo mount -t cifs //<tailscale-ip>/work /mnt/work -o user=RDP,uid=$(id -u)`
- macOS: Finder -> Go -> Connect to Server -> `smb://<tailscale-ip>/work`

**2. RDP drive redirection.** mstsc -> Show Options -> Local Resources -> Local devices and resources -> More -> tick drives/folders. They appear in the session as `\\tsclient\<drive>`.

**3. Clipboard.** Copy/paste files between your PC and the session in the RDP window.

Anything outside `C:\work` is lost when the job ends. Copy it into `C:\work` to get it in the artifact.

## What gets installed

- Embedded: ARM GCC, QEMU, VS Code, PlatformIO, pyserial
- Data: Python 3.11, JupyterLab, pandas, numpy, scipy, matplotlib
- Not preinstalled: Renode (install on demand), Wokwi (VS Code extension)

Start Jupyter: `jupyter lab --no-browser`, then open `http://localhost:8888` in the RDP session.

## Limits

- Max job time 360 min; workflow stops at 335 to leave time for the artifact upload.
- Disk is wiped when the job ends. Push to Git or download the artifact.
- Tools reinstall every run (~5-10 min).
- The `RDP` user is not an Administrator. Admin-level installs go in the "Install dev tools" step.
- Never store secrets or keys in `C:\work` (it is uploaded as an artifact).

## Security notes

- NLA enabled; RDP firewall rule scoped to `100.64.0.0/10` (Tailscale).
- Password comes from a secret and is never printed.
- Removed from the original: hardcoded password, admin rights, open port 3389, and the unverified remote `Downloads.bat` execution.

## ToS

GitHub Actions is for CI/CD. Using runners as a remote desktop can get a repo or account restricted. Use a private repo, keep sessions short, and move to a VM if you need this regularly.
