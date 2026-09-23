# 01 — Plan the lab

**Outcome:** every component has a role, a network path, and a private place to store deployment values.

## 1. Choose the boundaries

- **Windows host:** daily-use system. Install a Wazuh agent, Sysmon, and a harmless FIM test directory. Do not use it as a malware target.
- **Wazuh OVA:** local VM. Its dashboard is reachable from Windows on a local VMware network. Do not forward its ports through the home router.
- **T-Pot Standard VPS:** dedicated public host for honeypot traffic. Do not attach storage, accounts, or home subnet routes to it.
- **Tailscale:** private point-to-point path between the T-Pot VPS and the Wazuh VM. It does not make the VPS part of the home LAN.
- **Kali/pfSense VMs:** optional local-only attack simulation and firewall telemetry after the core integration works.

Record real addresses in a copy of [`inventory.example.md`](../inventory.example.md). Use names such as `wazuh-lab` and `tpot-vps` in both the Wazuh agent list and Tailscale console.

## 2. Reserve capacity

This Windows machine was checked with 32 GB RAM, an i7-12700K, and substantial free disk space. Start with **only the 8 GB Wazuh VM** while building the SIEM. A Kali VM and a pfSense VM can be started later and shut down when not in use. Reserve at least 100 GB of fast local storage for the OVA, snapshots, and optional VMs. The T-Pot VPS needs its own storage; it does not use local RAM.

## 3. Decide retention and costs before ordering

T-Pot's current Standard / Hive guidance calls for about **16 GB RAM and 256 GB SSD**. It includes its own Elastic stack. Confirm your provider's VPS cost, bandwidth rules, public IPv4 availability, and firewall controls before ordering. Decide how long to keep raw events in Wazuh and T-Pot. T-Pot's default raw-log persistence and index lifecycle are 30 days; change them only after measuring disk growth. Wazuh's local 50 GB OVA can fill sooner with high-volume honeypot events, so start with only Cowrie and Suricata alerts and inspect disk weekly.

## 4. Prepare private storage

Create a private evidence folder outside this Git repository. Save VM snapshots, raw logs, PCAPs, credentials, captured files, and the private inventory there. Git should contain **config templates and sanitized examples** only. Use a password manager for Wazuh, VPS, Tailscale, T-Pot, and VirusTotal credentials.

## 5. Define a test window

Record the UTC start and end time for every controlled test. Testing T-Pot from your own Windows/Kali machine is authorized because it is your VPS; keep all other active tests inside your own lab. A public honeypot will also receive unrelated internet traffic, so filter by the test source IP and timestamp during validation.

**Checkpoint:** you can name each host, its role, the path its events take to Wazuh, your VPS budget, and where private evidence will be stored.
