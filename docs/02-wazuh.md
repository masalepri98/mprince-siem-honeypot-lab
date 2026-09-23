# 02 — Import and secure Wazuh locally

**Outcome:** the Wazuh manager, indexer, and dashboard run in one VMware VM, reachable from Windows but not from the public internet.

## 1. Download the official OVA

Use the current [Wazuh virtual machine page](https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html) to download the OVA and published SHA512 checksum. Verify the file before import in PowerShell:

```powershell
Get-FileHash -Algorithm SHA512 'C:\path\to\wazuh.ova'
```

Compare the entire hash with Wazuh's published value. As of 2026-09-23 the documented OVA was 4.14.7, based on Amazon Linux 2023, with 4 vCPU, 8 GB RAM, and 50 GB disk. Record the version and hash in your private inventory.

## 2. Import into VMware Workstation

Open VMware Workstation → **File → Open** → choose the OVA → name it `wazuh-lab`. Confirm 4 vCPU and 8 GB RAM. Give it a local SSD location. Take a snapshot after first boot and another after the Windows agent is connected.

For the first setup, choose one of these local network layouts:

| Layout | Use when | Access to dashboard |
| --- | --- | --- |
| **Bridged** | Your router permits a VM to get its own LAN IP | Windows browser to `https://<WAZUH_LAN_IP>` |
| **VMware NAT** | Bridged networking is unavailable | Windows host to the VM's NAT address, if VMware's host adapter can reach it |

Verify reachability from Windows before continuing. Do not port-forward Wazuh TCP 443, 1514, 1515, 9200, or 55000 from your home router. Later, Tailscale on the VM lets the VPS agent connect without a router change.

## 3. Boot and check services

Log into the OVA console with the **current credentials stated on the Wazuh OVA page**. The documented defaults at writing were `wazuh-user` / `wazuh` for the OS and `admin` / `admin` for the dashboard. Find the VM IP and service status:

```bash
ip -brief address
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager
sudo ss -lntp | grep -E ':(443|1514|1515)\b'
```

From Windows:

```powershell
Test-NetConnection <WAZUH_LAN_IP> -Port 443
Test-NetConnection <WAZUH_LAN_IP> -Port 1514
Test-NetConnection <WAZUH_LAN_IP> -Port 1515
```

Open `https://<WAZUH_LAN_IP>` in your browser. The OVA uses a self-signed certificate by default; inspect its fingerprint and follow your browser's documented local-lab process for trusting it. Do not treat a certificate warning on an unrelated site as a lab step.

## 4. Replace defaults and stabilize the address

Change the OS password with `passwd`. Change the Wazuh dashboard/indexer admin password using [Wazuh's password management procedure](https://documentation.wazuh.com/current/user-manual/user-administration/password-management.html); its tool updates dependent all-in-one components. Save credentials in a password manager. Confirm the new dashboard login before closing the existing session. Assign a DHCP reservation for the VM or set a static address so the Windows agent does not lose its manager after a reboot.

## 5. Record a baseline

Record the OVA version, local IP, network mode, service status, and a dashboard screenshot in your private evidence folder. Take a VMware snapshot named `wazuh-baseline-secured`.

**Checkpoint:** Windows reaches the dashboard and ports 1514/1515; all three Wazuh services run; default credentials no longer work. The dashboard remains local-only.
