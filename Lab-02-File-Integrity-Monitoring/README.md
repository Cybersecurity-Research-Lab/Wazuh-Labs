# Lab 02: File Integrity Monitoring (FIM) Alert Investigation

## Scenario

Wazuh's File Integrity Monitoring (FIM / syscheck) module watches specified directories and alerts when files inside them are added, modified, or deleted. In this lab, a file inside a sensitive system directory (`/etc`) is modified outside of a normal patch window — the kind of alert that could mean anything from a routine config change by an admin to an attacker planting a backdoor or modifying access controls.

The goal is to walk through how to tell the difference using only the alert data available, since FIM alerts alone don't tell you *who* made the change or *why* — only *what* changed and *when*.

## Files in this lab

- [`sample-alerts.json`](./sample-alerts.json) — sanitized Wazuh FIM alert output for this scenario
- [`walkthrough.md`](./walkthrough.md) — full investigation, field by field

## How to reproduce this yourself

FIM in Wazuh requires directories to be explicitly configured for monitoring in `ossec.conf` on the agent:

```xml
<syscheck>
  <directories check_all="yes" realtime="yes">/etc</directories>
</syscheck>
```

Restart the agent after adding this, then modify a file inside that directory to trigger an alert:

```bash
sudo echo "# test change" >> /etc/hosts.allow
```

Check the Wazuh dashboard under **Integrity Monitoring** — you should see the change appear within seconds if `realtime="yes"` is set.

## What you'll learn

- How Wazuh's FIM module reports file changes (added/modified/deleted, hash diffs, permission changes)
- Why a file *hash* change matters more than the fact that "a file changed"
- How to distinguish a routine config edit from a suspicious one using timing, file path sensitivity, and process context
- Which Wazuh rule group governs FIM alerts and how severity is assigned
