# VPS selection for T-Pot Standard / Hive

**Research date:** 2026-09-23. Prices, promotions, location fees, and available OS images can change. Verify the final order summary before paying. This is a purchasing comparison, not a record of a VPS already ordered or tested.

## The minimum to shop for

This project runs **T-Pot Standard / Hive** on the VPS, including its own Elastic stack and dashboard. [T-Pot's requirements](https://github.com/telekom-security/tpotce#system-requirements) list **16 GB RAM and 256 GB SSD** for Hive, plus a public IPv4 address and an unrestricted outbound internet connection. Choose a minimal, currently supported Linux release; at this review, Debian 13 and Ubuntu 26.04 were on T-Pot's list. The VPS also needs a provider firewall so management ports can stay private while Cowrie listens publicly.

An 8 GB / 128 GB *Sensor* VPS is not equivalent: T-Pot Sensor expects a separate T-Pot Hive and does not supply the standalone dashboard used by this guide. A provider's 16 GB plan with only 160–200 GB of disk also falls short of the Hive storage baseline.

## Shortlist

| Plan | Published resources | Published price | Fit for this project |
| --- | --- | --- | --- |
| [Contabo Cloud VPS 8, Core](https://contabo.com/en-us/vps/) | 8 shared vCPUs, 24 GB RAM, 300 GB SSD; IPv4 and provider firewall included | Around **€14/month list**; **€11.20/month** is a first-24-month promotional rate. A US location adds a variable fee. | **Best budget starting point** for a month or more. Meets the RAM/disk baseline; SSD and shared CPU performance need a real T-Pot trial. |
| [Contabo Cloud VPS Plus 8, Performance](https://contabo.com/en-us/vps-performance/) | 8 AMD EPYC vCPUs, 24 GB RAM, 450 GB NVMe | Around **€35/month list**; **€28/month** first-24-month promotional rate, plus any location fee. | Better headroom for T-Pot's Elastic stack if the Core plan proves slow. |
| [DigitalOcean Basic Regular Droplet](https://www.digitalocean.com/pricing/droplets) | 8 shared vCPUs, 16 GiB RAM, 320 GiB SSD | **$0.14286/hour**, capped at **$96/month** for the bundled plan. | Good for a short, disposable test. A 48-hour run is about $6.86 before extras; a full month costs much more. [Debian 13 is an available image](https://docs.digitalocean.com/products/droplets/details/images/). |
| [IONOS VPS XL+](https://www.ionos.com/servers/vps) | 8 vCores, 16 GB RAM, 480 GB NVMe | **$11/month for three months** with a **one-year term**, then the page shows **$44/month**. | Meets the resource baseline but the advertised discount is a poor match for a short lab. |
| [Hostinger KVM 8](https://www.hostinger.com/vps-hosting) | 8 vCPUs, 32 GB RAM, 400 GB NVMe | Advertised **$25.99/month** promotional rate; page says renewal is **$49.99/month** for two years. Plans are paid upfront for the selected term. | Plenty of resources, but check total upfront and renewal costs rather than the displayed monthly equivalent. |

Sources above are the providers' own pricing pages. Contabo's [product documentation](https://docs.contabo.com/docs/products/cloud-vps/) says a one-month initial contract is available, includes a dedicated IPv4 and built-in firewall, and notes that non-EU regions can cost extra. Its [location-fee help page](https://help.contabo.com/en/support/solutions/articles/103000269774/) confirms US locations carry a surcharge; it does not publish one fixed amount for every plan. Confirm the exact monthly total and OS image in checkout.

## Recommendation for this lab

Start with **one month of Contabo Cloud VPS 8 Core**, choosing the 300 GB SSD option and a minimal T-Pot-supported Linux image. Select US Central if its quoted location fee is acceptable; the honeypot does not require low latency to the Windows machine because only logs cross the Tailscale link. Buy only a one-month term while testing. Check the provider's current terms or obtain provider confirmation that an intentionally exposed honeypot and stored attack samples are acceptable. Contabo's [general usage guidance](https://help.contabo.com/en/support/solutions/articles/103000271238-are-there-any-restrictions-on-the-content-allowed-on-my-server-) does not explicitly name T-Pot and requires a prompt response to abuse notices.

For a weekend-only proof of concept, DigitalOcean can cost less because of hourly billing, but **destroying** the Droplet ends compute billing; simply powering it off does not. See [DigitalOcean's billing documentation](https://docs.digitalocean.com/products/droplets/details/pricing/). A retained snapshot costs extra. For the several-week capture period likely needed to gather portfolio evidence, Contabo's monthly plan is the cheaper published choice.

## Check before ordering

1. Final month-to-month total, currency, tax, US location fee, setup fees, and cancellation date.
2. A minimal **Debian 13** or **Ubuntu 26.04** image, or a supported custom-image path. Do not install T-Pot on an older prebuilt application image.
3. Dedicated public IPv4, provider firewall, and permitted inbound Cowrie TCP 22 (optionally 23). Keep real SSH TCP 64295 restricted and dashboard TCP 64297 closed publicly.
4. Outbound access for package repositories, container images, Tailscale, and Wazuh agent traffic.
5. Provider's written policy for honeypots and abuse handling. Do not assume that a general VPS allowance explicitly covers captured malware samples.
6. Whether snapshots/backups and bandwidth are included, capped, or separately charged.

## First-day acceptance test

Before committing to a longer term, install T-Pot and verify that Cowrie, Suricata, Kibana, and T-Pot's web UI start; the log path is writable; and the VPS survives one reboot. Check `free -h`, `df -h`, `docker stats`, and `systemctl status tpot`. Run the guide's controlled SSH test, then inspect the Cowrie JSON and Kibana event. If Elasticsearch or Logstash repeatedly crashes, disk fills quickly, or the system stalls under modest traffic, move to a faster plan or reduce enabled services before relying on it for portfolio evidence.
