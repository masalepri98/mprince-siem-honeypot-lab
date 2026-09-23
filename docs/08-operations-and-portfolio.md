# 08 — Operate, troubleshoot, and publish

## Routine checks

| Frequency | Check |
| --- | --- |
| Each lab session | Wazuh and T-Pot service status; agent connectivity; one fresh Cowrie event |
| Weekly during public exposure | VPS and OVA disk (`df -h`), index growth, T-Pot log persistence, VPS provider traffic/billing, updates |
| Before upgrades | Snapshot/backup, current versions and Git revisions, custom rule and config backups |
| After upgrades or a daily T-Pot reboot | Agent reconnected, JSON path unchanged, fresh test event crosses the pipeline |

T-Pot's default raw-log persistence and Elastic index lifecycle are 30 days. Wazuh's OVA is only 50 GB by default. Tune retention and filter noisy events before increasing storage. Do not use a public repository as a raw log archive.

## Troubleshooting by hop

| Symptom | First checks |
| --- | --- |
| Wazuh dashboard not ready | `systemctl status wazuh-indexer wazuh-manager wazuh-dashboard`; free RAM/disk; indexer logs |
| Windows agent missing | Local manager IP, TCP 1514/1515, `WazuhSvc`, Windows agent log |
| T-Pot management unavailable | Provider console, provider firewall, `systemctl status tpot`, TCP 64295/64297, tailnet grants |
| Cowrie gets no public traffic | Provider firewall TCP 22, `docker ps`, `ss -lntp`, correct VPS public IP |
| Cowrie JSON has events but Wazuh has none | Actual `TPOT_DATA_PATH`, file permissions, agent `ossec.log`, tailnet TCP 1514, agent status |
| Wazuh receives events but custom alert absent | `wazuh-logtest` phase 2/3, `eventid` field, rule ID collision, manager restart |
| Suricata absent | Confirm Standard compose enables Suricata, actual EVE path, `event_type=alert`, agent localfile path |
| pfSense syslog absent | Isolated NIC IPs, UDP 514 listener, `allowed-ips`, pfSense remote log destination and selected categories |
| VirusTotal absent | FIM event first, private API key, `integrations.log`, API rate limit |

For T-Pot service issues use [its troubleshooting section](https://github.com/telekom-security/tpotce#troubleshooting). For Wazuh agent issues use [agent enrollment troubleshooting](https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/troubleshooting.html). Record the exact failed hop before changing multiple settings.

## Recovery and shutdown

Back up the **Wazuh VM**, T-Pot VPS snapshot/config, private inventory, custom rules, and sanitized evidence. Test a VM restore by starting a copy on an isolated network, not alongside the live VM. If you stop the VPS project, take a final sanitized export, remove the T-Pot node from the tailnet and Wazuh agent list, close its provider firewall rules, and cancel the VPS so billing stops. Keep the writeup and code after shutdown; the live lab is not required for a useful portfolio entry.

## Portfolio case study

The existing site at [masonprince93.com](https://masonprince93.com/) uses Hugo/PaperMod and has both Posts and Projects. Publish a full post, then link it from Projects. Use [`templates/hugo-post.md`](../templates/hugo-post.md) as a starting outline. Hugo [page bundles](https://gohugo.io/content-management/page-bundles/) let `content/posts/siem-honeypot-lab/index.md` live beside sanitized screenshots and diagrams.

The post should answer:

1. What question did the lab test? For example, *Can an internet-facing Cowrie event reach a local SIEM without exposing the SIEM?*
2. What architecture and constraints did you choose?
3. What did you configure yourself? Link the custom Wazuh rules and the tailnet policy design.
4. How did you test each hop? Show the same event ID/session in raw Cowrie JSON, T-Pot Kibana, and Wazuh.
5. What did you observe? Include latency, counts, one detection-tuning choice, and one failure you fixed.
6. What remains limited? For example, Suricata visibility depends on the VPS traffic and ruleset; pfSense logs represent the isolated local network, not the VPS perimeter.

Only publish results actually measured after deployment. Redact public IPs, usernames, captured passwords, tokens, private paths, and unsolicited attacker content. Treat captured files and transcripts as untrusted; do not upload them to the website or GitHub. Keep aggregate counts and timestamps only when they cannot identify third parties.

**Final checkpoint:** a reader can follow the repository, reproduce the architecture, and distinguish your real observations from planned tests. The portfolio post links to the repository and contains sanitized evidence from the running lab.
