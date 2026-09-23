# 07 — Add the rest of the SIEM lab

The core Wazuh + T-Pot integration should pass chapter 06 first. These extensions reproduce the useful parts of the referenced SOC lab without creating a second Wazuh deployment.

## A. Isolated pfSense/Kali network and firewall logs

Use a **VMware LAN Segment** or equivalent host-only isolated VM network, for example `10.77.10.0/24`. Give pfSense two NICs: WAN on VMware NAT for updates, LAN on the isolated segment (`10.77.10.1/24`). Give Kali one NIC on the isolated segment. Add a second NIC to the Wazuh VM on that segment (`10.77.10.10/24`) with **no default gateway**; keep Wazuh's original NIC as its management/internet route. Snapshot before changing NICs. Never bridge Kali or pfSense LAN to your home router during attack simulation.

On the Wazuh VM, identify the *new* NIC with `nmcli device status`. Assign `10.77.10.10/24` to that NIC using the Amazon Linux network configuration or NetworkManager; set `ipv4.never-default yes`. Confirm the original Wazuh LAN IP, dashboard, and Tailscale remain reachable after the change. On Kali, use pfSense's LAN DHCP and confirm gateway `10.77.10.1`.

Wazuh's [syslog receiver](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/syslog.html) is disabled by default. In the manager's `/var/ossec/etc/ossec.conf`, add this **additional** remote block inside `<ossec_config>`; keep the existing secure 1514 block:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.77.10.1/32</allowed-ips>
  <local_ip>10.77.10.10</local_ip>
</remote>
```

Restart Wazuh manager and check `sudo ss -lnup | grep ':514'`. In pfSense, open **Status → System Logs → Settings → Remote Logging**, enable remote logging, set destination `10.77.10.10:514`, and select firewall and system/authentication events. Create a harmless pfSense LAN rule and use Kali to generate a permitted and a blocked connection to your own lab target. Compare pfSense's log with Wazuh's received event. pfSense sends basic syslog over UDP **without encryption**, which is acceptable only on this isolated local VM segment; do not send it over the internet. See [Netgate's remote logging guide](https://docs.netgate.com/pfsense/en/latest/monitoring/logs/remote.html).

This pfSense network is for **local** attack simulation. It does not sit between the public internet and the T-Pot VPS. T-Pot's own Suricata watches the VPS traffic; installing Suricata on the Windows host would not automatically see packets exchanged solely between VMware guests.

## B. VirusTotal enrichment on FIM alerts

Obtain a VirusTotal API key for your own account. Keep it only in the Wazuh manager's local configuration and your password manager. Wazuh's [official VirusTotal integration](https://documentation.wazuh.com/current/user-manual/capabilities/malware-detection/virus-total-integration.html) queries **hashes** from FIM events; it does not upload the file contents through this integration.

Inside `/var/ossec/etc/ossec.conf` on the Wazuh manager, add:

```xml
<integration>
  <name>virustotal</name>
  <api_key>REPLACE_LOCALLY_WITH_PRIVATE_API_KEY</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

Restart the manager. Add or modify a **benign** file under `C:\LabEvidence\FIM-Test` and look for the FIM alert followed by a VirusTotal response. A new file may have **no record** in VirusTotal; that still proves the lookup path. Check `/var/ossec/logs/integrations.log` for rate limits or errors. Do not commit the manager's populated `ossec.conf` or the API key. Remove the integration when finished if you do not intend to keep making lookups.

## C. Wazuh investigation views

Create saved searches or dashboard panels for:

- Cowrie event count by `eventid` and UTC hour.
- Failed versus simulated successful Cowrie logins.
- Top source IPs **in the private dashboard only**; redact or aggregate for the portfolio.
- Cowrie command events and file download events.
- T-Pot Suricata alert signatures.
- Windows Sysmon and FIM alerts by agent.
- pfSense allows/blocks, if that extension is enabled.

Take one screenshot of the finished view with time range and filters visible. Export saved objects privately for backup; review them before publishing because saved objects can contain addresses, queries, and environment names.

**Checkpoint:** each extension has its own source event, Wazuh destination event, and a recorded explanation of the relationship. The Wazuh manager's existing agent transport still works after adding pfSense syslog.
