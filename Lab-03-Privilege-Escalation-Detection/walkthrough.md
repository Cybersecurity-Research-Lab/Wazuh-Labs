# Walkthrough: Privilege Escalation Investigation

## Step 1: The first alert looks routine

The first entry in [`sample-alerts.json`](sample-alerts.json) — rule `80792`, level 3 — just says a command was executed via sudo. This is extremely common in any environment; admins run `sudo` commands constantly. On its own, level 3 barely registers as a low-priority informational event.

The two fields that matter here are `auid` and `euid`:

- `auid` (audit user ID) — the **original logged-in user**, which stays constant even through privilege changes. Here it's `1001` (`labuser`).
- `euid` (effective user ID) — the **current effective privilege**, which changes as commands run with elevated rights. Here it's `0` (`root`), because sudo itself always runs as root momentarily to execute the permitted command.

This is expected: sudo is *designed* to briefly run commands as root. Nothing suspicious yet.

## Step 2: The second alert is where it gets interesting

Rule `80793`, level 8, two seconds later: a new process (`/bin/bash`) spawned with `euid=0`, but this time the `parent_process` is `vim` — not `sudo` directly. This is the critical detail.

Vim was granted sudo access (in this lab's deliberately misconfigured setup) to let `labuser` edit a specific file as root. But vim has a built-in `:!` command that shells out to an arbitrary command — which inherits vim's *current* privilege level. So instead of using vim to edit a file, `labuser` used vim's permitted execution to spawn a full root **bash shell**, completely outside the intended scope of the sudo rule.

This is the essence of a sudo-misconfiguration privilege escalation: the sudo rule wasn't wrong in granting *some* elevated access — it was wrong in granting access to a binary (`vim`) that can be abused to obtain unrestricted root access, rather than a narrowly scoped one.

## Step 3: The correlated alert

Rule `80799`, level 12, ties it together explicitly: the same `auid` (`labuser`, still logged in as themselves) obtained an `euid=0` shell within seconds of using a sudo-permitted binary. This is Wazuh's audit correlation recognizing the *auid staying constant while euid escalates via an indirect path* — a strong, reliable signal for this class of attack, independent of which specific binary was abused (vim, less, find, awk, and several others share this same shell-out weakness).

## Step 4: Fields to check before concluding

| Field | Why it matters |
|---|---|
| `auid` vs `euid` | `auid` never changes for a session — if it stays the same while `euid` jumps to 0, you're looking at a privilege escalation within that same logged-in session, not a separate root login |
| `parent_process` | Is the elevated process's parent the expected binary (e.g. `sudo` directly), or something else (like `vim`, `less`, `find`) known for shell-out capability? |
| Command executed | Was it the narrowly-scoped command the sudo rule intended, or something broader like `/bin/bash`? |
| Time between sudo execution and shell spawn | A near-immediate jump (seconds) from permitted execution to a shell strongly suggests deliberate abuse rather than coincidence |
| Was this expected? | Check whether `labuser` has a legitimate reason to need root — and whether the sudo rule granting vim access was ever supposed to allow this |

## Step 5: Verdict

Given:
- `auid` stayed constant (`labuser`) while `euid` jumped to `0`
- The escalation path went through a known shell-out-capable binary (`vim`), not a direct sudo call for `bash`
- The timing (2 seconds) rules out coincidence
- No indication this user should have unrestricted root access

This is a confirmed **privilege escalation via sudo misconfiguration** — a true positive, and one of the more common real-world root causes flagged in security audits (GTFOBins-style abuse of overly broad sudo grants). The appropriate response: revoke the offending sudoers rule immediately, review what the user did with the root shell (check bash history, files touched), and audit all other sudoers entries on the host for similar shell-out-capable binaries (`vim`, `less`, `awk`, `find`, `man`, and others).

## Step 6: The rule logic behind it

Unlike Labs 01 and 02, this detection relies on `auditd` integration rather than FIM or plain log parsing — Wazuh needs process-level visibility (`execve` syscalls, `auid`/`euid` tracking) to catch this class of attack. This is worth noting for anyone building automated triage on top of Wazuh: **not all attack classes are visible through the same data source**, and a triage system needs to know which alert types come with enough context (like `auid`/`euid` pairs) to reason about automatically versus which need a human to pull additional logs.

## Takeaway

Privilege escalation doesn't always mean an exploit — very often it's a misconfigured permission being used exactly as configured, just not as *intended*. The `auid`/`euid` relationship is the single most reliable signal for catching this pattern, regardless of which specific binary or technique was used.
