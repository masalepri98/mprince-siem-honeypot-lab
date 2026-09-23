# 06 — Rules and end-to-end validation

**Outcome:** a controlled Cowrie event goes from public VPS → T-Pot JSON → Wazuh agent → manager rule → dashboard. A Suricata event follows the same path when available.

## 1. Install the local Cowrie rules

Copy [`config/wazuh/tpot_rules.xml`](../config/wazuh/tpot_rules.xml) to `/var/ossec/etc/rules/tpot_rules.xml` **on the Wazuh VM**. The OVA has a text-only console, so use SSH from Windows for ordinary copy/paste and `scp` for files. The dashboard `admin` account is different from the VM's `wazuh-user` account. From Windows PowerShell, with the repo cloned locally:

```powershell
Set-Location 'C:\Users\mprin\Documents\PT_upskillin\mprince-siem-honeypot-lab'
scp .\config\wazuh\tpot_rules.xml wazuh-user@<WAZUH_LAN_IP>:/home/wazuh-user/tpot_rules.xml
ssh wazuh-user@<WAZUH_LAN_IP>
```

Enter the **VM operating-system password** when prompted. In the SSH session, place the file under Wazuh's custom rules directory:

```bash
sudo install -o root -g root -m 0644 ~/tpot_rules.xml /var/ossec/etc/rules/tpot_rules.xml
```

Check that IDs `100500–100505` do not collide with other custom rules. The rules match Cowrie's documented `eventid` values using Wazuh's built-in JSON decoder; no custom decoder is required for these fields.

Before restarting the manager, test each sample line from [`samples/cowrie-events.jsonl`](../samples/cowrie-events.jsonl):

```bash
sudo /var/ossec/bin/wazuh-logtest
```

Paste **one JSON line at a time**. Confirm phase 2 says `json` and phase 3 reports rule `100501`, `100502`, then `100503`. The example IP `198.51.100.25` is reserved for documentation. Test the repeat-failure rule by pasting the failed-login event repeatedly in the same `wazuh-logtest` session until rule `100505` fires; verify the count and timing because Wazuh's correlation behavior depends on the active ruleset and session. If the rule syntax fails, use the error location to correct it before restarting the manager.

```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```

Wazuh [custom rules](https://documentation.wazuh.com/current/user-manual/ruleset/rules/custom.html) under `/var/ossec/etc/rules` survive package upgrades. Cowrie [documents its event codes](https://docs.cowrie.org/en/latest/OUTPUT.html). The repeat-failure rule groups events by Cowrie's dynamic `src_ip` field; change its threshold after observing real traffic and false positives.

## 2. Generate a controlled end-to-end event

From Windows or Kali, connect to **your own VPS public IP**:

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no root@<VPS_PUBLIC_IP>
```

If prompted, enter a **throwaway lab string**, never a real password. Stop after a login attempt or a harmless command such as `id`. Record the UTC timestamp and the IP you tested from. Depending on Cowrie's configuration, the attempt may be marked failed or successful; match the actual `eventid` rather than assuming one.

On the VPS:

```bash
cd ~/tpotce
data_path="$(realpath "$(sed -n 's/^TPOT_DATA_PATH=//p' .env | tail -n 1)")"
tail -n 10 "$data_path/cowrie/log/cowrie.json"
sudo tail -n 30 /var/ossec/logs/ossec.log
```

In the T-Pot native Kibana dashboard, find the same `session` and timestamp. In Wazuh Threat Hunting, filter on `agent.name: tpot-vps` and the rule ID or `eventid`. The source IP, username, and timestamp should agree across the raw JSON, Kibana, and Wazuh. Save **redacted** screenshots and record the latency from T-Pot log timestamp to Wazuh alert timestamp. Do not publish captured passwords, full session transcripts, or actual public IPs.

## 3. Confirm Suricata collection

On the VPS, find the actual EVE file under `${TPOT_DATA_PATH}/suricata/log`, then locate a JSON line with `"event_type":"alert"`. T-Pot Standard includes Suricata, but a specific alert depends on traffic and the active ruleset. Wazuh has [built-in Suricata JSON support](https://documentation.wazuh.com/current/proof-of-concept-guide/integrate-network-ids-suricata.html). Compare the Suricata `signature`, source/destination, and timestamp in the file with the Wazuh alert. If no alert exists yet, record that result and revisit after traffic produces one; do not claim success from a service-health check alone.

## 4. Build a cross-source investigation

Choose one bounded time window. Show:

1. The Cowrie login or command event in T-Pot.
2. The matching Wazuh alert on `tpot-vps`.
3. A Windows Sysmon/FIM event from the local endpoint in the same Wazuh dashboard.
4. If enabled, pfSense firewall logs from the isolated VM network.

Explain which events are causally related and which merely share a time window. A public Cowrie hit has no reason to create a Windows Sysmon event unless you deliberately generate a separate local test.

## 5. Acceptance record

| Check | Expected evidence |
| --- | --- |
| Source | One Cowrie JSON line with eventid, session, timestamp |
| Native T-Pot view | Same event in Kibana |
| Agent | `tpot-vps` Active in Wazuh; no collection error |
| Rule | Wazuh rule 100501/100502/100503/100504; 100505 for repeated failures |
| Suricata | One real EVE alert and corresponding Wazuh alert, when available |
| Transport | Wazuh ports reachable only over tailnet/local network; no public home port forward |

**Checkpoint:** one controlled Cowrie event is traceable through **all four** stages with matching values. A rule test alone is not an end-to-end pass.
