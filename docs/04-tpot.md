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

If the VPS image starts an SMTP service such as Exim on TCP 25, T-Pot will stop before making changes because its honeypots need that port. On a dedicated honeypot VPS that does not send local mail, identify the listener and disable only that mail service. This leaves SSH running:

```bash
sudo ss -ltnp '( sport = :25 )'
sudo systemctl disable --now exim4
sudo ss -ltnp '( sport = :25 )'  # no listener means the port is free
```

If the listener is not Exim, identify its systemd unit with `sudo systemctl list-sockets` or `sudo ss -lntup` before changing it. Disabling Exim also stops local mail delivery, including any system alerts sent through it.

## 2. Install T-Pot Standard / Hive

Connect to the new host using its initial SSH port. Inspect the current [T-Pot installation instructions](https://github.com/telekom-security/tpotce#installation) and supported OS list. On a minimal Debian host, as the regular sudo-capable user, install `tmux` so the interactive installer keeps running if SSH disconnects. **Run from that user's home**, not `/root/tpotce`:

```bash
sudo apt-get update
sudo apt-get install -y git curl tmux
cd "$HOME"
git clone https://github.com/telekom-security/tpotce.git
cd ~/tpotce
git rev-parse HEAD
tmux new -s tpot-install
# In the tmux shell:
./install.sh
```

If `~/tpotce` is already a valid clone, skip `git clone`. If an earlier attempt was run from `/root/tpotce`, leave that copy alone and use the clone in your regular user's home. You can detach from `tmux` with **Ctrl+B**, then **D**. If SSH drops before the reboot, reconnect on TCP 64295 and run `tmux attach -t tpot-install` to see the installer. A reboot ends the tmux session; after reboot, check T-Pot's service status instead. Keep the provider console available for recovery. Do not put the T-Pot web password on the command line or in shell history.

If the Ansible playbook stops at its first task with `Duplicate become password prompt` and `Sorry, try again`, the **BECOME password** was rejected by `sudo`. It is the Linux password for the account running `./install.sh`, not the Contabo root password, SSH key passphrase, or T-Pot web password. Confirm it interactively without printing it:

```bash
sudo -k
sudo -v
sudo id -u  # expected: 0
sudo -k     # clear the cached authorization before rerunning the installer
```

If `sudo -v` fails, reset that user's password from the provider console using a root session (`passwd <OS_USER>`), then repeat the check. Do not set passwordless sudo just to bypass the error. Re-run `./install.sh` as the regular user and enter the verified password when Ansible asks for `BECOME password`. Ansible failed before its first playbook task, so this error alone does not require a VPS reinstall.

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
