# Walkthrough: File Integrity Monitoring Investigation

## Step 1: Look at what actually changed

The first alert in [`sample-alerts.json`](./sample-alerts.json) is rule `550`, "Integrity checksum changed," on `/etc/passwd` — one of the most sensitive files on a Linux system, since it defines system accounts.

The key fields here aren't just "a file changed," but *how*:

- `size_before` (2201) vs `size_after` (2249) — the file grew, consistent with a line being added (e.g. a new user account)
- `md5_before`/`md5_after` and `sha1_before`/`sha1_after` differ — confirms actual content change, not just a timestamp touch
- `uid_after`/`gid_after` are `0`/`0` — the file is still owned by root, so no ownership change, only content

A hash change on `/etc/passwd` outside of normal user-provisioning activity is worth investigating on its own. But the next alert is what makes this genuinely concerning.

## Step 2: The second alert, one second later

Rule `554`, "File added to the system," reports a brand-new file at `/etc/cron.d/systemd-update` — created **one second after** the `/etc/passwd` change. Two things stand out:

- The filename (`systemd-update`) is designed to look like a legitimate system file, but `/etc/cron.d/` entries are a well-known persistence mechanism — anything placed here runs on a schedule, as root, without a login being required again.
- The timing (1 second after a `/etc/passwd` change) is not coincidental for a routine admin task. A human running `useradd` doesn't also drop a new cron file the same second, unless it's the same automated action doing both.

## Step 3: The correlated alert

Rule `594`, level 12, ties the two events together explicitly: a `/etc/passwd` modification followed within a short window by a new file in `/etc/cron.d`. This is Wazuh's correlation logic recognizing a *pattern* consistent with an attacker adding a new account and establishing persistence via cron in the same operation — a common technique after gaining initial access.

## Step 4: Fields to check before concluding

| Field | Why it matters |
|---|---|
| `path` | Is this a sensitive system path (`/etc`, `/root`, `/bin`) or something low-risk (a log file, a temp directory)? |
| `md5_before`/`md5_after` (or `sha1`) | Confirms real content change vs. a metadata-only touch |
| `uid_after`/`gid_after`/`perm_after` | Did ownership or permissions change too? A permission change (e.g. `644` → `777`) on a sensitive file is an even stronger signal |
| Timing relative to other alerts | Multiple FIM events within seconds of each other, across related paths, suggests a single automated action rather than unrelated coincidences |
| Was this expected? | Cross-check against change management / patch schedule if your environment tracks that — a real admin change during a known maintenance window is a very different story than one at 2:47 AM with no ticket |

## Step 5: Verdict

Given:
- A content change to `/etc/passwd` with no corresponding ownership/permission anomaly on its own
- A new file appearing in a known persistence location one second later
- No known maintenance window at 02:47 UTC
- Wazuh's own correlation already flagging this combination at level 12

This pattern is a likely **true positive for attacker persistence activity**, not routine administration. The appropriate action in a real environment: isolate the host, review the new cron file's contents immediately, check `/etc/passwd` for any newly added account, and pull authentication logs for the window leading up to `02:47:11` to identify how access was obtained in the first place.

## Step 6: The rule logic behind it

Wazuh's base FIM rules (`550`–`594` range) fire on checksum, permission, and existence changes to monitored paths. The value of the correlated rule (`594` in this sample) is exactly the same principle as Lab 01: individual FIM events are common and often benign, but Wazuh's correlation engine — or a well-designed AI triage layer sitting on top of it — earns its value by recognizing *combinations* of low-severity events that together indicate something serious.

## Takeaway

A single file change is rarely the full story. The sequence and proximity of changes across related paths is what separates "someone edited a config" from "someone is establishing persistence." This is a pattern worth watching for across every FIM-based investigation, not just this scenario.
