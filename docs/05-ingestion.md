# 05 — Private T-Pot → Wazuh ingestion

**Outcome:** the T-Pot VPS sends Cowrie and Suricata events to the local Wazuh manager without exposing the manager through the home router.

## 1. Create a dedicated tailnet policy

Use a dedicated Tailscale tailnet for this lab if you do not already manage one. Before connecting the VPS, replace the tailnet's broad default grant with a narrow policy. [`config/tailscale/grants.example.json`](../config/tailscale/grants.example.json) allows the tagged T-Pot node to initiate **only TCP 1514 and 1515** to the tagged Wazuh node; the admin's devices may reach management ports. Adapt the admin selector to your account and validate the policy in the Tailscale console. Tailscale [grants](https://tailscale.com/docs/features/access-control/grants) are additive: a pre-existing `*:*` rule would still allow broader access.

Do **not** enable Tailscale subnet routing, exit-node service, SSH sharing, or public Funnel on the T-Pot VPS. The goal is a single agent-to-manager path.

## 2. Install Tailscale on the Wazuh OVA

The official Wazuh OVA is Amazon Linux 2023. From its shell, use [Tailscale's Amazon Linux 2023 package instructions](https://dl.tailscale.com/stable/):

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://pkgs.tailscale.com/stable/amazon-linux/2023/tailscale.repo
sudo yum install -y tailscale
sudo systemctl enable --now tailscaled
sudo tailscale up --advertise-tags=tag:lab-siem
tailscale ip -4
```

Complete authentication at the URL shown by `tailscale up`. Record the tailnet IPv4 privately. Verify Wazuh still works locally:

```bash
sudo systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard
sudo ss -lntp | grep -E ':(1514|1515)\b'
```

## 3. Install Tailscale on the T-Pot VPS

Install it **after** T-Pot succeeds so its installer sees a minimal system. For a supported Debian VPS, follow [Tailscale's Linux instructions](https://tailscale.com/docs/install/linux):

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-tags=tag:lab-tpot
tailscale ip -4
tailscale ping <WAZUH_TAILNET_IP>
```

Complete the authentication. From the VPS, verify only the intended manager services are reachable:

```bash
nc -vz <WAZUH_TAILNET_IP> 1514
nc -vz <WAZUH_TAILNET_IP> 1515
```

If `nc` is absent, install `netcat-openbsd` or use `bash`'s TCP test. Check the Tailscale console's grant policy and the Wazuh VM firewall before changing any public firewall rules. On the VPS, `ip route` should have **no route to your home LAN through Tailscale**. Once tailnet management works, close public TCP 64295 at the VPS provider firewall and manage SSH through its tailnet address. Install Tailscale on Windows too if you want the native T-Pot web UI at `https://<TPOT_TAILNET_IP>:64297` without an SSH tunnel.

## 4. Install and enroll the Wazuh agent on T-Pot

Use the [current Wazuh Linux agent instructions](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html). On a Debian-based VPS:

```bash
sudo apt-get update
sudo apt-get install -y gnupg apt-transport-https curl
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo 'deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main' | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt-get update
sudo env WAZUH_MANAGER='<WAZUH_TAILNET_IP>' WAZUH_REGISTRATION_SERVER='<WAZUH_TAILNET_IP>' WAZUH_AGENT_NAME='tpot-vps' apt-get install -y wazuh-agent
sudo systemctl enable --now wazuh-agent
```

Use an agent version **no newer than the manager**. If current packages have moved ahead of the OVA, install the matching package version from Wazuh's package list or upgrade the manager first. Wazuh agent traffic uses TCP 1514 and automatic enrollment uses TCP 1515. Confirm `tpot-vps` appears Active in Wazuh Agents management.

## 5. Add T-Pot JSON files to the agent

Resolve the data path and inspect actual filenames on the VPS:

```bash
cd ~/tpotce
data_path="$(realpath "$(sed -n 's/^TPOT_DATA_PATH=//p' .env | tail -n 1)")"
printf 'T-Pot data path: %s\n' "$data_path"
find "$data_path/cowrie/log" -maxdepth 1 -type f -name '*.json' -ls
find "$data_path/suricata/log" -maxdepth 1 -type f -name '*.json' -ls
namei -l "$data_path/cowrie/log/cowrie.json"
```

`TPOT_DATA_PATH` is often `./data`, relative to `~/tpotce`; the resolved value printed above is the path Wazuh needs. `namei -l` reveals traversal permissions, but do not make captured logs world-readable. In `/var/ossec/etc/ossec.conf` on the **VPS agent**, add the two blocks from [`config/wazuh/tpot-agent-localfile.xml`](../config/wazuh/tpot-agent-localfile.xml) inside the existing `<ossec_config>` element. Replace `__TPOT_DATA_PATH__` with the **absolute** path printed above, for example `/home/<OS_USER>/tpotce/data`. Include the Suricata block only after confirming `eve.json` exists.

Do not point Wazuh at T-Pot's Elasticsearch index or `/var/lib/docker/containers`. The persistent JSON files under `TPOT_DATA_PATH` are the integration boundary. T-Pot continues to maintain its own Kibana indices.

Restart and inspect:

```bash
sudo /var/ossec/bin/wazuh-logcollector -t
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent --no-pager
sudo tail -n 50 /var/ossec/logs/ossec.log
```

If the localfile is unreadable, identify the actual logcollector process account with `ps -eo user,args | grep '[w]azuh-logcollector'`, then grant that account narrow directory traversal and file-read ACLs for the required log paths. Do not loosen permissions on the whole T-Pot tree or expose captured downloads.

## 6. Reboot test

After the path works, reboot the VPS. Confirm T-Pot, Tailscale, and Wazuh agent all return; confirm the agent re-registers as **the same** Wazuh agent and resumes new Cowrie events. T-Pot may reboot daily by default, so this is part of acceptance, not an optional check.

**Checkpoint:** the VPS agent is Active, no home-router Wazuh ports are forwarded, the localfile paths exist and are readable, and the path survives a VPS reboot.
