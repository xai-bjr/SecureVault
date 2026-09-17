# SecureVault / HomeServer

**A self-hosted, encrypted file server you run on your own hardware — reachable from anywhere through a zero-trust Tailscale tunnel, with no ports opened to the public internet.**

![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-2.x-000000?logo=flask&logoColor=white)
![Tailscale](https://img.shields.io/badge/Tailscale-Zero--Trust%20VPN-3d3d3d?logo=tailscale&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue)
![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20Linux%20%7C%20macOS-lightgrey)
![Status](https://img.shields.io/badge/Status-Active-4ee08a)

---

## Contents

1. [What this is](#what-this-is)
2. [Prerequisites & System Requirements](#prerequisites--system-requirements)
3. [Windows 11 Setup](#windows-11-setup)
4. [Linux Setup (Ubuntu / Mint / AntiX / Debian)](#linux-setup-ubuntu--mint--antix--debian)
5. [Environment Variables (`.env`)](#environment-variables-env)
6. [Tailscale Remote Access](#tailscale-remote-access)
7. [Troubleshooting & Common Errors](#troubleshooting--common-errors)
8. [Scale & Performance Analysis](#scale--performance-analysis)
9. [Project Structure](#project-structure)

---

## What this is

SecureVault is a Flask-based file server that stores everything on a drive you physically own — `D:\Storage` on Windows or `/mnt/storage` on Linux/macOS — and exposes it only over a [Tailscale](https://tailscale.com) mesh VPN. There is no public-facing port, no third-party cloud bucket, and no account you don't control. Passwords are hashed with **Argon2id**; sessions use short-lived, per-login tokens; the long-lived request-signing key lives only in your local `.env` file.

See `index.html` (included in this repo) for a visual architecture blueprint of the full request path.

---

## Prerequisites & System Requirements

| Requirement | Minimum | Notes |
|---|---|---|
| **Python** | 3.10+ | 3.11 or 3.12 recommended; check with `python --version` |
| **pip** | Bundled with Python 3.10+ | Upgrade with `python -m pip install --upgrade pip` |
| **RAM** | 2 GB free | 4 GB+ recommended if also running Tailscale + a reverse proxy |
| **Disk** | Depends on your storage volume | The app itself needs <100 MB; storage capacity is whatever you mount |
| **Network** | Outbound internet for initial setup (pip, Tailscale) | No inbound public ports required after setup |
| **Tailscale account** | Free tier is sufficient | One account, one tailnet, as many devices as you like |
| **OS** | Windows 11, Ubuntu 20.04+/Mint 21+/AntiX/Debian 11+, or macOS 12+ | Instructions below cover Windows and Linux in detail |

---

## Windows 11 Setup

### 1. Install Python and clone the project

```powershell
# Verify Python is installed and on PATH
python --version

# Clone or copy the project, then move into it
cd C:\Projects
git clone https://github.com/yourname/securevault.git
cd securevault
```

### 2. Create and activate a virtual environment

```powershell
python -m venv venv
venv\Scripts\activate
```

Your prompt should now show `(venv)` at the start of the line. If PowerShell blocks the activation script with an `ExecutionPolicy` error, run this once (as your normal user, not admin):

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### 3. Install dependencies

```powershell
pip install -r requirements.txt
```

`requirements.txt` should include at minimum:

```
Flask>=2.3
python-dotenv>=1.0
argon2-cffi>=23.1
waitress>=2.1
```

> **Why Waitress and not Gunicorn?** Gunicorn relies on `os.fork()`, which does not exist on Windows. Waitress is a pure-Python, production-grade WSGI server that runs natively on Windows and is the correct choice here — Gunicorn is used on the Linux side below.

### 4. Set up your storage directory

```powershell
mkdir D:\Storage
```

### 5. Create your `.env` file

```powershell
Copy-Item .env.example .env
notepad .env
```

**Important:** Windows Explorer hides known file extensions by default, so `Copy-Item .env.example .env` can silently save as `.env.txt` if you create it through Explorer's "New > Text Document" instead. Confirm the real filename with:

```powershell
Get-ChildItem -Force | Select-Object Name
```

You should see `.env`, not `.env.txt`. See [Troubleshooting](#troubleshooting--common-errors) if it's wrong.

### 6. Open the port in Windows Defender Firewall

**Option A — PowerShell/CMD (`netsh`):**

```powershell
netsh advfirewall firewall add rule name="SecureVault" dir=in action=allow protocol=TCP localport=8443
```

**Option B — GUI:**

1. Open **Windows Defender Firewall with Advanced Security** (search from the Start menu).
2. Click **Inbound Rules** → **New Rule…** in the right-hand panel.
3. Select **Port** → **Next**.
4. Select **TCP**, then **Specific local ports**, and enter `8443` (or `8080` for the non-TLS dev port) → **Next**.
5. Select **Allow the connection** → **Next**.
6. Leave all three profiles (Domain, Private, Public) checked unless you specifically want to restrict it → **Next**.
7. Name it `SecureVault` → **Finish**.

> Because remote access goes through Tailscale (see [below](#tailscale-remote-access)), you generally only need this rule to allow traffic on Tailscale's own virtual network adapter — you do **not** need to forward this port on your home router.

### 7. Run it

```powershell
# Development (single-threaded, auto-reload, NOT for continuous use)
python app.py

# Production (Waitress, multi-threaded, binds to all interfaces)
waitress-serve --host=0.0.0.0 --port=8443 app:app
```

### 8. (Optional) Run it as a background service on boot

Windows has no built-in equivalent to systemd, so the common approaches are:

- **NSSM** (Non-Sucking Service Manager) — wraps `waitress-serve` as a proper Windows Service:
  ```powershell
  nssm install SecureVault "C:\Projects\securevault\venv\Scripts\waitress-serve.exe" "--host=0.0.0.0 --port=8443 app:app"
  nssm start SecureVault
  ```
- **Task Scheduler** — create a task triggered "At log on" or "At startup" that runs the same `waitress-serve` command with "Run whether user is logged on or not" checked.

---

## Linux Setup (Ubuntu / Mint / AntiX / Debian)

### 1. Install system packages

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git ufw
```

### 2. Clone the project and create a virtual environment

```bash
git clone https://github.com/yourname/securevault.git
cd securevault

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

`requirements.txt` on Linux additionally includes Gunicorn instead of Waitress:

```
Flask>=2.3
python-dotenv>=1.0
argon2-cffi>=23.1
gunicorn>=21.2
```

### 3. Set up your storage mount

```bash
sudo mkdir -p /mnt/storage
sudo chown $USER:$USER /mnt/storage
```

If `/mnt/storage` is a separate physical drive, add it to `/etc/fstab` so it survives reboots — check its UUID first with `sudo blkid`, then add a line like:

```
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /mnt/storage  ext4  defaults  0  2
```

### 4. Create your `.env` file

```bash
cp .env.example .env
nano .env
```

### 5. Run it manually (sanity check first)

```bash
gunicorn --bind 0.0.0.0:8443 --workers 2 app:app
```

Open `http://127.0.0.1:8443` on the same machine to confirm it responds before wiring up the service below.

### 6. Install it as a systemd service

Create `/etc/systemd/system/securevault.service`:

```ini
[Unit]
Description=SecureVault Flask file server
After=network.target tailscaled.service

[Service]
User=youruser
Group=youruser
WorkingDirectory=/home/youruser/securevault
EnvironmentFile=/home/youruser/securevault/.env
ExecStart=/home/youruser/securevault/venv/bin/gunicorn --bind 0.0.0.0:8443 --workers 2 app:app
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Replace `youruser` with your actual Linux username in all three places. Then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now securevault
sudo systemctl status securevault
```

### 7. Configure `ufw`

```bash
sudo ufw allow 8443/tcp
sudo ufw enable
sudo ufw status verbose
```

As on Windows, this rule only matters for traffic arriving over the Tailscale interface (`tailscale0`) if you keep your router's port-forwarding untouched — see the next section.

---

## Environment Variables (`.env`)

Copy `.env.example` to `.env` and fill in real values. **Never commit `.env` to version control** — add it to `.gitignore`.

```ini
# ---------------------------------------------------------------------------
# SecureVault environment configuration
# ---------------------------------------------------------------------------

# Long, random secret used to sign session tokens.
# Generate one with: python -c "import secrets; print(secrets.token_hex(32))"
VAULT_SECRET_KEY=replace-this-with-64-hex-characters

# Host binding.
# 127.0.0.1  = only reachable from this machine (safe default for local testing)
# 0.0.0.0    = reachable from any interface, including Tailscale's virtual NIC
#              (required for remote access — see Troubleshooting if this is wrong)
VAULT_HOST=0.0.0.0

# Port the Flask app listens on.
# 8080 = plain HTTP, convenient for local dev
# 8443 = conventionally used for HTTPS/TLS in this project
VAULT_PORT=8443

# Absolute path to the storage root.
# Windows example:  D:\Storage
# Linux example:    /mnt/storage
VAULT_STORAGE_PATH=/mnt/storage

# Absolute path where access/error logs are written.
VAULT_LOG_DIR=/mnt/storage/.logs

# Argon2id hashing cost parameters (higher = slower to brute-force, slower to log in).
# Defaults below are a reasonable balance for a home server CPU.
VAULT_ARGON2_TIME_COST=3
VAULT_ARGON2_MEMORY_COST_KB=65536
VAULT_ARGON2_PARALLELISM=2

# Set to "production" to disable Flask's debug reloader/traceback pages.
VAULT_ENV=production
```

---

## Tailscale Remote Access

This is what lets your phone on a 4G/5G connection, or your laptop on an untrusted coffee-shop Wi-Fi, reach a server sitting on your home network — without opening a single port to the public internet.

### 1. Install Tailscale on the server

- **Windows:** download and run the installer from [tailscale.com/download](https://tailscale.com/download).
- **Linux:**
  ```bash
  curl -fsSL https://tailscale.com/install.sh | sh
  sudo tailscale up
  ```

### 2. Authenticate

Running `tailscale up` (or launching the Windows app) opens a browser link the first time — log in with the same account you'll use on every other device. This links the machine to your private **tailnet**.

### 3. Note the server's Tailscale IP

```bash
tailscale ip -4
```

This returns an address in the `100.x.x.x` range — this is the address your other devices will use, and it stays stable even as your home IP or mobile carrier IP changes.

### 4. Install Tailscale on your client devices

- **Phone (iOS/Android):** install the Tailscale app from the App Store/Play Store, sign in with the same account.
- **Laptop:** same installer as above, same account.

### 5. Connect

Once both the server and the client show as connected in `tailscale status`, browse to:

```
https://100.x.x.x:8443
```

from the client device — over Wi-Fi, 4G, 5G, or any other internet connection. Tailscale builds a direct encrypted (WireGuard) tunnel between the two devices whenever possible, and falls back to a relay only when a direct path can't be established (e.g. restrictive carrier-grade NAT).

### 6. (Recommended) Restrict which devices can reach the server

In the [Tailscale admin console](https://login.tailscale.com/admin/machines), you can:
- Rename the server's machine entry to something memorable (e.g. `securevault-home`)
- Set up **ACLs** (Access Control Lists) to restrict which tailnet devices are allowed to reach port 8443 on the server, rather than trusting every device on the tailnet by default

---

## Troubleshooting & Common Errors

| Symptom | Likely Cause | Fix |
|---|---|---|
| `ERR_CONNECTION_REFUSED` in browser | Server process isn't running, or is bound to the wrong host/port | Confirm the process is running (`systemctl status securevault` / check the PowerShell window); confirm `VAULT_PORT` in `.env` matches the URL you're typing |
| `ERR_CONNECTION_REFUSED` only from **other devices**, works on `localhost` | App is bound to `127.0.0.1` instead of `0.0.0.0` | Set `VAULT_HOST=0.0.0.0` in `.env` and restart the app — `127.0.0.1` only accepts connections that originate from the same machine |
| `.env` settings seem to be ignored entirely | The file was actually saved as `.env.txt` by Windows Explorer | Run `Get-ChildItem -Force` in the project folder; if you see `.env.txt`, rename it: `Rename-Item .env.txt .env` |
| `OSError: [Errno 98] Address already in use` / `[WinError 10048]` | Another process already has that port open | Linux: `sudo lsof -i :8443` to find the PID, then `kill <PID>`. Windows: `netstat -ano \| findstr :8443`, then `taskkill /PID <pid> /F` |
| Works on the LAN, but not over Tailscale from a mobile network | Firewall rule only allows the LAN adapter, not the Tailscale virtual adapter | Re-check the `netsh`/`ufw` rule was added for the **port**, not scoped to a specific network interface — the rules above are interface-agnostic by design |
| `tailscale status` shows the server as offline from a client's perspective | `tailscaled` service isn't running, or the device hasn't authenticated | Linux: `sudo systemctl status tailscaled`; re-run `sudo tailscale up`. Windows: check the Tailscale system-tray icon shows "Connected" |
| Login always fails even with the correct password | `argon2-cffi` version mismatch between the environment that created the hash and the one verifying it, or the hash was truncated by a database/column-length limit | Reinstall pinned versions with `pip install -r requirements.txt --force-reinstall`; confirm the stored hash column allows at least 97 characters (Argon2id's typical encoded length) |
| Large file uploads time out | Default Flask/Gunicorn/Waitress timeouts are too short for very large files on a slow link | Increase `--timeout` on Gunicorn (`gunicorn --timeout 120 ...`) or Waitress's `channel_timeout`, and raise Flask's `MAX_CONTENT_LENGTH` if it's capping uploads |
| Systemd service fails immediately with `status=203/EXEC` | `ExecStart` path in the unit file doesn't match your actual venv/username | Double check every absolute path in `securevault.service` matches `whoami` and the real clone location, then `sudo systemctl daemon-reload` |

---

## Scale & Performance Analysis

### Resource usage at rest

| Resource | Idle | Under load (active transfer) |
|---|---|---|
| **CPU** | <1% on any modern CPU (Flask/Gunicorn/Waitress workers are event-driven, not busy-polling) | Spikes during Argon2id hashing on login (by design — this is deliberately expensive) and during large file I/O |
| **RAM** | ~40–80 MB per worker process | Grows modestly with concurrent upload/download buffers; not proportional to file size, since files are streamed rather than loaded fully into memory |
| **Bandwidth** | Near-zero — Tailscale's keepalive traffic is tiny | Bounded by your weakest link: home upload speed, the client's connection, or disk I/O, whichever is slowest |

### How concurrent file reads/writes are handled

- Each HTTP request is handled by a separate **worker** (Gunicorn `--workers N` on Linux, Waitress's internal thread pool on Windows) — two clients downloading different files run genuinely in parallel, bounded by CPU cores and disk throughput, not by the Python process itself.
- File writes should stream to disk in chunks (e.g. reading `request.stream` in a loop) rather than buffering the entire upload in memory — this keeps RAM usage flat regardless of whether someone uploads a 10 MB photo or a 10 GB archive.
- For the **personal/home scale** (1–5 devices), 2 Gunicorn workers or Waitress's default thread pool is more than sufficient — you are almost always disk- or network-bound, not CPU-bound.
- At **small-business scale** (5–25 users), bump Gunicorn to `--workers 4` and monitor `iostat`/Resource Monitor during peak hours (typically backup windows) to confirm disk I/O, not the app itself, is the bottleneck.
- At **enterprise scale** (100+ users), a single disk and a single app process are no longer the right shape for the problem — that's the point at which the architecture in this README's Scale 3 column (reverse proxy, containerized horizontal scaling, S3/MinIO object storage) becomes necessary rather than optional. See `index.html`'s Scale Matrix tab for the full breakdown.

---

## Project Structure

```
securevault/
├── app.py                  # Flask application entry point
├── requirements.txt        # Pinned Python dependencies
├── .env.example             # Template — copy to .env and fill in real values
├── .env                     # Your real config (gitignored, never commit this)
├── index.html               # Visual architecture blueprint dashboard (this repo)
├── venv/                    # Virtual environment (gitignored)
└── securevault.service      # Example systemd unit file (Linux)
```

---

## License

MIT — use it, fork it, self-host it. See `LICENSE` for the full text.
