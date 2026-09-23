---
title: "Building a Local Wazuh SIEM with a Remote T-Pot Honeypot"
date: 2026-09-23
draft: true
tags: ["wazuh", "honeypot", "t-pot", "detection-engineering"]
categories: ["Projects"]
summary: "A tested path from a public honeypot event to a local SIEM alert."
---

> Draft template: replace every bracketed field with observed facts before publishing.

## Question and outcome

[State the question. Summarize what worked and the date tested.]

## Architecture

[Insert a sanitized diagram. Explain the T-Pot VPS, private transport, Wazuh OVA, and Windows endpoint.]

## What I built

[Versions, selected components, why this network layout, and links to the configuration in the repo.]

## Test and evidence

| Stage | Evidence | UTC timestamp |
| --- | --- | --- |
| Cowrie JSON | [redacted eventid/session] | [measured] |
| T-Pot Kibana | [sanitized screenshot] | [measured] |
| Wazuh agent/rule | [rule ID and screenshot] | [measured] |
| Suricata/pfSense/Windows | [only if tested] | [measured] |

## Detection tuning

[Original rule or adjustment, why the threshold was chosen, test cases and observed false positives.]

## What broke and how I fixed it

[One concrete failure, the evidence that identified its hop, and the fix.]

## Limits and next steps

[Where the lab differs from production. What you would measure or change next.]

## Reproduce it

[Link to the public GitHub repository and exact tested commit.]
