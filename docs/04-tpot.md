# 04 — Deploy standalone T-Pot on a VPS

**Outcome:** a dedicated VPS runs T-Pot Standard / Hive, serves Cowrie on public TCP 22/23, and has a restricted native dashboard. This phase does not yet connect it to Wazuh.

## 1. Select and provision the VPS

Use a VPS with at least **16 GB RAM, 256 GB SSD, public IPv4**, a provider firewall, and a **currently supported minimal T-Pot Linux distribution**. The [upstream requirements](https://github.com/telekom-security/tpotce#system-requirements) are authoritative. At writing, Debian 13 Network Install and Ubuntu 26.04 Live Server were listed. A provider image with extra web servers, DNS services, or desktop packages can conflict with honeypot ports; start from a minimal image with SSH. Record provider, region, OS release, billing limits, and public IP in your private inventory.

Create a dedicated SSH key in PowerShell, then add **only its `.pub` file contents** to the VPS during provisioning. Keep the private key out of Git:

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\tpot_lab_ed25519" -C "tpot-lab"
Get-Content "$env:USERPROFILE\.ssh\tpot_lab_ed25519.pub"
```

Use the provider firewall as the first boundary:

| Port | Source | Purpose |
| --- | --- | --- |
| TCP 22 | Public | Cowrie SSH honeypot after installation |
| TCP 23 | Public, optional | Cowrie Telnet honeypot |
| TCP 64295 | Your current admin IP only | Real OS SSH after T-Pot moves it |
| TCP 64297 | Closed publicly | T-Pot web UI; access via Tailscale later |
| Other honeypot ports | Add selectively | Only for enabled services you are studying |

Allow outbound HTTPS, DNS, and package/image downloads. **Before starting the installer, pre-allow TCP 64295 from your admin IP.** The installer moves real SSH there, while TCP 22 becomes a honeypot. Keep the provider console/recovery access available in case SSH changes unexpectedly. Do not open Wazuh ports 1514/1515 on the VPS or home router.

## 2. Install T-Pot Standard / Hive

Connect to the new host using its initial SSH port. Inspect the current [T-Pot installation instructions](https://github.com/telekom-security/tpotce#installation) and supported OS list. On a minimal Debian host with `git` and `curl` installed, as the regular sudo-capable user:

```bash
git clone https://github.com/telekom-security/tpotce.git ~/tpotce
cd ~/tpotce
git rev-parse HEAD
./install.sh
```

Choose **Standard / Hive** when prompted, create the T-Pot web user, review the installer's stated changes, and reboot as directed. Do not choose **Sensor**: upstream requires a separate T-Pot Hive for that mode, whereas this architecture sends events to Wazuh.

After reboot, use the real SSH port:

```powershell
ssh -i "$env:USERPROFILE\.ssh\tpot_lab_ed25519" -p 64295 <OS_USER>@<VPS_PUBLIC_IP>
```

On the VPS:

```bash
sudo systemctl status tpot --no-pager
docker ps --format 'table {{.Names}}\t{{.Status}}'
grep '^TPOT_DATA_PATH=' ~/tpotce/.env
cd ~/tpotce
data_path="$(realpath "$(sed -n 's/^TPOT_DATA_PATH=//p' .env | tail -n 1)")"
printf 'T-Pot data path: %s\n' "$data_path"
find "$data_path/cowrie/log" -maxdepth 1 -name 'cowrie.json' -ls
```

The current Standard compose file mounts Cowrie logs under `${TPOT_DATA_PATH}/cowrie/log`, and Cowrie writes `cowrie.json` there. Resolve the actual `TPOT_DATA_PATH` from `.env` before adding any Wazuh configuration. T-Pot also stores Suricata logs under `${TPOT_DATA_PATH}/suricata/log` when that service is enabled. Confirm the actual EVE filename with `find`; do not assume a path merely because this guide mentions one.

## 3. Verify the honeypot and native dashboard

From your own Windows machine, make a single SSH connection to **your own VPS** on TCP 22. It should reach Cowrie, not the OS SSH daemon. Exit after a harmless login attempt. On the VPS, inspect the last events:

```bash
cd ~/tpotce
data_path="$(realpath "$(sed -n 's/^TPOT_DATA_PATH=//p' .env | tail -n 1)")"
tail -n 5 "$data_path/cowrie/log/cowrie.json"
```

T-Pot's landing page is HTTPS on TCP 64297. Keep it closed in the provider firewall. Until Tailscale is connected, use a restricted SSH local tunnel from Windows if needed:

```powershell
ssh -i "$env:USERPROFILE\.ssh\tpot_lab_ed25519" -p 64295 -L 64297:127.0.0.1:64297 <OS_USER>@<VPS_PUBLIC_IP>
```

Open `https://localhost:64297` and verify the T-Pot landing page and Kibana. The certificate may be self-signed; inspect it as you would any management UI. Later you can use the VPS tailnet address for management. T-Pot's [remote access documentation](https://github.com/telekom-security/tpotce#remote-access-and-tools) describes the ports and accounts.

## 4. Set a baseline

Record the T-Pot Git revision, selected compose file, service health, log paths, and provider firewall rules. Take a VPS snapshot if your provider offers one. Decide whether to keep T-Pot's default community data submission; [upstream explains the opt-out](https://github.com/telekom-security/tpotce#community-data-submission). Do not commit captured payloads or attacker credentials.

**Checkpoint:** real SSH is on 64295, Cowrie responds on 22, a Cowrie JSON event is present, and the T-Pot native dashboard works through restricted management access.
