# Walkthrough: SSH Brute Force Investigation

## Step 1: Start with the individual alerts

Looking at [`sample-alerts.json`](./sample-alerts.json), the first three entries all share the same rule ID (`5710`, "Attempt to login using a non-existent user") and severity level 5. On their own, none of these would justify paging anyone — a single failed login is completely normal, whether it's a typo or a bot scanning the internet at large.

What matters is the pattern across entries, not any single one:

- Same `srcip` (`203.0.113.44`) across all attempts
- Different `srcuser` values each time (`admin`, `root`, `test`) — this is a strong signal. A real user mistyping their password uses *their own* username repeatedly. An attacker cycling through common usernames is testing which accounts exist.
- Timestamps only 2–3 seconds apart — far faster than a human typing

## Step 2: The correlated alert

The fourth entry is different: rule `5720`, level 10, "Multiple authentication failures from same source IP." This isn't a raw log line — it's Wazuh's correlation engine recognizing that several level-5 events from the same source crossed a threshold (by default, 8 failures in a short window) and rolling them into a single higher-severity alert.

This is the alert that should actually trigger analyst attention. The jump from level 5 to level 10 reflects that: isolated failures are noise, a *pattern* of failures is a signal.

## Step 3: Key fields to check

When you get an alert like this in a real environment, pull these fields before concluding anything:

| Field | Why it matters |
|---|---|
| `srcip` | Is this IP known-bad (check threat intel), internal, or a cloud/VPN range? |
| `agent.name` | Which host is being targeted — is it internet-facing (expected to get scanned) or internal (more concerning if internal)? |
| `srcuser` values across the burst | Cycling through many different usernames = automated guessing. Repeated same username = more likely a real user with a wrong password or an expired credential. |
| Timing | Sub-second to few-second intervals between attempts strongly suggests automation, not a human. |
| Was there a *successful* login after the failures? | This is the single most important follow-up — check rule `5715` (successful login) or similar from the same `srcip` shortly after. If present, treat this as a likely compromise, not just a blocked attempt. |

## Step 4: Verdict

Based on the fields above:
- **Multiple usernames, single source IP, sub-3-second intervals, no successful login following** → classic automated brute-force scanning. Very likely a **true positive** for "brute-force attempt," but with **no evidence of compromise** since there's no successful auth in the sample.
- Action in a real environment: confirm no successful login occurred, then block/rate-limit the source IP and check whether other hosts saw traffic from the same IP (lateral scanning).

If the data instead showed a successful login (`5715`) immediately after the failures from the same IP, the verdict would change entirely — that would warrant an active incident response, not just a blocked-IP note.

## Step 5: The rule behind it

Wazuh's default ruleset already includes this correlation logic (rule `5720`, part of the `sshd` rule group), built on top of the individual `5710`-series rules. It's a good example of why raw log monitoring alone isn't enough — the value is in Wazuh's ability to correlate many low-severity events into one meaningful alert, which is exactly the kind of reasoning an AI SOC assistant would need to replicate (and explain) if it's going to be trusted with real alert triage.

## Takeaway

The individual failed-login alerts are not the story. The story is in the pattern: one IP, many usernames, tight timing, no success. That distinction — single event vs. correlated pattern — is the core skill this lab is meant to build.
