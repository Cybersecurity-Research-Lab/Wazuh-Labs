# Lab 01: SSH Brute Force Detection

## Scenario

A monitored Linux server is being targeted by repeated SSH login attempts from a single external IP address. Most attempts use common or guessed usernames. This is one of the most frequent alert types a SOC analyst encounters — and also one of the easiest to misjudge, since a handful of failed logins can be a legitimate typo, while a burst of dozens in seconds is almost always automated.

The goal of this lab is to walk through the raw alert data and reason through the investigation the way an analyst would, not just note that "brute force = bad."

## Files in this lab

- [`sample-alerts.json`](./sample-alerts.json) — sanitized Wazuh alert output for this scenario
- [`walkthrough.md`](./walkthrough.md) — full investigation, field by field

## How to reproduce this yourself

On the monitored endpoint (with the Wazuh agent installed, see [00-setup](../00-setup/minimal-setup-for-labs.md)), simulate the attack from a separate machine using `hydra` or a simple loop:

```bash
for i in {1..10}; do
  ssh baduser@<monitored-endpoint-ip> -o ConnectTimeout=2
done
```

You should see Wazuh generate a series of authentication failure alerts, escalating to a correlated "multiple authentication failures" alert once the threshold is crossed (default: 8 failures within a short window, rule group `authentication_failures`).

## What you'll learn

- How Wazuh correlates repeated low-severity events into a single higher-severity alert
- Which fields distinguish a real brute-force attempt from noise (failed logins from legitimate users, misconfigured scripts, etc.)
- How to pull the source IP, targeted usernames, and timing pattern out of raw alert JSON
- Which Wazuh rule ID is responsible and how its logic works
