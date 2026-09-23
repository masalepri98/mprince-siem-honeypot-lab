# Deployment checklist

Check a box only after recording evidence of the result. Keep evidence in a private folder until it is sanitized for publication.

## Local foundation

- [ ] Record host RAM, CPU, free SSD space, VMware version, and a restore point or backup.
- [ ] Import the current Wazuh OVA; assign at least 4 vCPU, 8 GB RAM, and 50 GB disk.
- [ ] Give the OVA a stable local address; verify Windows can reach its dashboard and TCP 1514/1515.
- [ ] Change the OVA's default OS and dashboard credentials; store them in a password manager.
- [ ] Snapshot the working Wazuh VM.
- [ ] Enroll the Windows host and verify it is active.
- [ ] Install Sysmon, collect its Operational channel, and prove a test process event arrived.
- [ ] Enable FIM on a test directory and prove a created/modified file alert arrived.

## VPS and private path

- [ ] Choose VPS with supported minimal OS, at least 16 GB RAM, 256 GB SSD, public IPv4, and provider firewall.
- [ ] Create and secure VPS account/SSH key. Restrict management TCP 64295; allow selected honeypot ports.
- [ ] Install T-Pot Standard / Hive; record its Git revision and container image versions.
- [ ] Confirm Cowrie is listening and `cowrie.json` contains a test event.
- [ ] Confirm the T-Pot native web UI and Kibana work through restricted management access.
- [ ] Join Wazuh VM and T-Pot VPS to Tailscale with narrow grants; verify T-Pot cannot reach your home subnet.
- [ ] Install Wazuh agent on VPS with manager tailnet address and enroll it.
- [ ] Configure agent collection for Cowrie JSON and verified Suricata EVE path.
- [ ] Confirm the T-Pot agent remains active after a VPS reboot.

## Detection and publication

- [ ] Install and test the custom Cowrie rules with `wazuh-logtest`.
- [ ] Generate a Cowrie test event against your own VPS and prove it appears in both dashboards.
- [ ] Generate or observe a Suricata alert and prove Wazuh ingested it.
- [ ] Add pfSense/syslog and VirusTotal only after the core event path works.
- [ ] Build a Wazuh dashboard view for Cowrie event type, source IP, and timeline.
- [ ] Record latency, event counts, retention settings, limitations, and one tuning decision.
- [ ] Sanitize evidence and publish a case study on masonprince93.com.
