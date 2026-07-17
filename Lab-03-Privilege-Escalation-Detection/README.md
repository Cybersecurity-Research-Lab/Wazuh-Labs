# Lab 03: Privilege Escalation Detection

## Scenario

A low-privileged user on a monitored Linux server runs a command that results in a root shell — the classic outcome of a successful privilege escalation. In this lab, the technique is a misconfigured `sudo` rule that allows a low-privilege user to run a binary with unintended root-level capabilities (a common real-world misconfiguration, distinct from a memory-corruption exploit).

Privilege escalation alerts are high-stakes: they mean an attacker (or a legitimate but overly-broad permission) just crossed from limited access to full control of a host. The goal of this lab is to learn how to read process-level and auditd-style alert data to confirm what actually happened, rather than reacting to the alert title alone.

## Files in this lab

- [`sample-alerts.json`](sample-alerts.json) — sanitized Wazuh alert output for this scenario
- [`walkthrough.md`](walkthrough.md) — full investigation, field by field

## How to reproduce this yourself

This requires Wazuh's integration with Linux audit (`auditd`) for process-level visibility, since FIM alone won't catch this.

On the monitored agent:

```bash
sudo apt install auditd -y
sudo systemctl enable --now auditd
```

Add an audit rule to watch for `sudo` execution and privilege changes:

```bash
sudo auditctl -a always,exit -F arch=b64 -S execve -F euid=0 -F auid!=0 -k privilege_escalation
```

Set up a deliberately over-permissive sudo rule for a test user (for lab purposes only, never do this on a real system):

```bash
echo "labuser ALL=(ALL) NOPASSWD: /usr/bin/vim" | sudo tee /etc/sudoers.d/labuser-test
```

Then, as `labuser`, escalate using vim's shell-out capability:

```bash
sudo vim -c ':!/bin/bash'
```

This spawns a root shell. Check the Wazuh dashboard for the resulting alert.

## What you'll learn

- How Wazuh surfaces privilege escalation via `auditd` integration, not just FIM or log parsing
- How to read `auid` (audit user ID, the *original* logged-in user) vs `euid` (effective user ID) to detect a privilege jump
- Why sudo misconfigurations are one of the most common real-world escalation paths — more common in practice than kernel exploits
- How to distinguish a legitimate admin using sudo normally from an escalation event
