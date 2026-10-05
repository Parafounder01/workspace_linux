# 🖥️ Windows RDP over Tailscale — On-Demand GitHub Actions Runner

> Spin up a fresh Windows Server 2022 box, reach it via RDP over your private Tailscale network, use it, and let it tear itself down. No public exposure, no standing infrastructure cost.

---

## 🚀 What this does

| Step | Action |
|---|---|
| 1️⃣ | GitHub Actions spins up a `windows-2022` runner |
| 2️⃣ | Enables RDP **with NLA enforced** (encrypted auth, not disabled) |
| 3️⃣ | Firewall rule scoped to **Tailscale's CGNAT range only** (`100.64.0.0/10`) — never open to the public internet |
| 4️⃣ | Creates a throwaway `RDP` user with a **random 20-char password**, masked in logs, **not** an Administrator |
| 5️⃣ | Installs Tailscale and joins as an **ephemeral node** (auto-removes from your tailnet when the run ends) |
| 6️⃣ | Verifies port 3389 is reachable, prints connection info |
| 7️⃣ | Stays alive for the session window, then tears the Tailscale node down (`if: always()`) |

---

## 🔧 Setup

1. Add a repo secret: **Settings → Secrets and variables → Actions**
   - `TAILSCALE_AUTHKEY` — a [reusable, ephemeral auth key](https://tailscale.com/kb/1085/auth-keys) from your Tailscale admin console.
2. Make sure your own device is on the same tailnet.
3. Commit `rdp-tailscale.yml` under `.github/workflows/`.

## ▶️ Run it

**Actions tab → "Windows RDP via Tailscale (ephemeral)" → Run workflow.**

Open the step **"Print connection info"** in the run log for:
```
Address : <tailscale IP>
Username: RDP
Password: <random, masked unless you're viewing your own run>
```

Connect with any RDP client (`mstsc`, Microsoft Remote Desktop, etc.) to that address.

## ⏱️ Session length

Default window: **330 minutes**, workflow timeout **350 minutes** (GitHub's hard cap is 360 for hosted runners). Cancel the run anytime from the Actions tab to end the session early — the Tailscale node is removed either way.

---

## 🔒 Why this version is safe to run

Compared to a "backdoor-RDP" script, this workflow deliberately avoids:

- ❌ Hardcoded passwords → ✅ random, masked, generated per run
- ❌ Admin-group backdoor user → ✅ `RDP` user is in **Remote Desktop Users** only
- ❌ NLA/encryption disabled → ✅ NLA **enforced**
- ❌ Firewall open to `any` → ✅ scoped to Tailscale's own address range
- ❌ Auto-downloading and silently executing a `.bat` from a random third-party URL at every boot → ✅ **no remote script execution, period**
- ❌ Persistent node left on your tailnet → ✅ `--ephemeral` flag, explicit `tailscale logout` teardown

## ⚠️ Still worth knowing

- Anyone with push access to this repo (or who can trigger `workflow_dispatch`) can view the RDP password in their own run's log. Treat repo write access accordingly.
- This spins up a cloud VM under GitHub's control — fine for short-lived test/dev access, not a replacement for a hardened, monitored jump box in production.
