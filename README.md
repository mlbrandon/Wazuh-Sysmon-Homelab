# Wazuh SIEM Homelab with Sysmon Endpoint Monitoring

A working SOC homelab built on my laptop with free tools: a **Wazuh SIEM** server collecting Windows Security and **Sysmon** logs from a Windows 11 endpoint, with a verified failed-logon detection test.

**Built:** October 2026 · **Author:** Miles Brandon, Cybersecurity student at Old Dominion University (CompTIA Security+)

## Why I built it

To get hands-on blue team experience beyond certification theory: deploy a SIEM, enroll an endpoint, forward logs, and prove that attack-like activity shows up where an analyst would look.

**GRC tie-in:** the lab implements **CIS Control 8 (Audit Log Management)**: collect, centralize, and review audit logs. It complements my [GRC portfolio](https://github.com/mlbrandon/grc-portfolio), where logging was assessed on paper.

## Architecture

```
Linux Mint host (ThinkPad P14s) - VirtualBox 7.2 - bridged network
|
|-- win11-endpoint (Windows 11 Enterprise Eval)
|     Sysmon (SwiftOnSecurity config) --+
|     Windows Security log -------------+--> Wazuh agent 4.14.9
|                                                  |
|                                       events (TCP 1514)
|                                                  v
|-- wazuh-server (Ubuntu Server 24.04)
|     Wazuh manager (rules, alerts) --> Wazuh indexer --> Wazuh dashboard
|
|-- Browser on Mint --HTTPS--> Wazuh dashboard
```

| Component | Specs |
| --- | --- |
| Host | ThinkPad P14s Gen 5, Linux Mint 22.2, VirtualBox 7.2 |
| wazuh-server | Ubuntu Server 24.04, 8 GB RAM, 4 CPUs, 50 GB disk, Wazuh 4.14 all-in-one |
| win11-endpoint | Windows 11 Enterprise Evaluation, 4 GB RAM, 2 CPUs, 64 GB disk, Wazuh agent 4.14.9 |

## What I built

1. **Virtualization host:** VirtualBox on Linux Mint, with KVM disabled so VirtualBox could use the CPU's VT-x.
2. **Wazuh server:** Ubuntu Server VM with the Wazuh all-in-one install (indexer, manager, dashboard).
3. **Windows endpoint:** Windows 11 VM with a local lab account and the Wazuh agent enrolled (status Active).
4. **Detection test:** 5 deliberate failed logons on the endpoint.
5. **Sysmon:** installed with the SwiftOnSecurity config and added the `Microsoft-Windows-Sysmon/Operational` channel to the agent's `ossec.conf`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

I took a VirtualBox snapshot before every risky step so I could roll back.

## Results

**Failed-logon detection:** 5 wrong passwords produced **exactly 5 alerts** (Wazuh rule 60122, Windows Event ID 4625).

![Agent active](screenshots/01-agent-active.png)
![Failed logon alerts](screenshots/02-failed-logons.png)
![Alert detail](screenshots/03a-alert-detail.png)
![Alert detail, rule fields](screenshots/03b-alert-detail.png)

**Sysmon ingestion:** Sysmon logged 31+ events locally, and the query `data.win.system.channel:"Microsoft-Windows-Sysmon/Operational"` returned **30 hits** in the Wazuh dashboard.

![Sysmon install](screenshots/04-sysmon-install.png)
![Sysmon in Event Viewer](screenshots/05-sysmon-eventviewer.png)
![ossec.conf Sysmon block](screenshots/06-ossec-conf-sysmon.png)
![Sysmon events in Wazuh](screenshots/07-sysmon-wazuh-events.png)

**Not yet proven:** detection of specific attacker behavior. A test `whoami` command was recorded by Sysmon but did not generate an alert under Wazuh's default rules. Collecting a log is not the same as alerting on it.

## Troubleshooting highlights

I hit and resolved 13 issues. The most useful ones:

| Problem | Root cause | Fix |
| --- | --- | --- |
| VM aborted: "cannot operate in VMX root mode" | KVM kernel modules held the CPU's VT-x | Unloaded and permanently blacklisted the KVM modules |
| Wazuh installer kept rolling back | Too little RAM, then a broken leftover package holding ports 1515 and 55000 | Raised RAM to 8 GB; rebuilt the VM clean after a force-purge failed |
| Logs stopped after changing the admin password | Filebeat still stored the old password | Updated the Filebeat keystore, restarted it, verified with `filebeat test output` |
| Mint kernel panic on boot | Broken initramfs for the newest kernel | Booted an older kernel from GRUB, rebuilt initramfs and GRUB |
| Windows snapshot failed: "Could not copy NVRAM file" | Bug in VirtualBox 7.0.16 (ruled out disk space and permissions first) | Upgraded to VirtualBox 7.2 from Oracle's repo |
| Sysmon log missing from Event Viewer | Event Viewer was open before the install | Confirmed the service was running, reopened Event Viewer |

## What I learned

- How logs flow from an endpoint to a SIEM: source log, agent, manager rules, indexer, dashboard.
- **Collection is not alerting.** Rules decide what becomes an alert, so tuning and custom rules matter.
- Rule out simple causes (resources, typos, a powered-off server) before blaming the tool.
- Snapshot before risky changes. A clean rebuild is sometimes faster than a repair.

## Next steps

- [ ] Write a custom Wazuh rule for discovery commands like `whoami`, then retest
- [ ] Simulate attacks with Atomic Red Team
- [ ] Map detections to MITRE ATT&CK
