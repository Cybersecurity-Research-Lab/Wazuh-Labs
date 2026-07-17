# Wazuh Labs

Hands-on log analysis and alert investigation labs built on [Wazuh](https://wazuh.com/), a free and open-source SIEM/XDR platform.

Most Wazuh tutorials online stop at installation. This repo picks up where they leave off: given a real alert, how do you actually investigate it like an analyst would?

## What's inside

Each lab presents a realistic attack scenario, the raw Wazuh alert(s) it generates, and a full investigation walkthrough — the questions an analyst should ask, how to tell a true positive from a false positive, and which detection rule fired and why.

| Lab | Scenario | Focus |
|---|---|---|
| [lab-01-brute-force-ssh](Lab-01-SSH-Brute-Force-Detection) | Repeated failed SSH logins | Authentication log analysis |
| [lab-02-file-integrity-monitoring](Lab-02-File-Integrity-Monitoring) | Unauthorized file modification | FIM alerts |
| [lab-03-privilege-escalation-detection](Lab-03-Privilege-Escalation-Detection) | Local privilege escalation attempt | Process/syscall analysis |
| [lab-04-malware-execution-alert](Lab-04-Malware-Execution-Alert) | Malicious binary execution | Alert correlation |

*(Labs are added incrementally — check back for updates.)*

## Lab format

Every lab follows the same structure:

1. **Scenario** — what's happening, in plain language
2. **Raw alert(s)** — sanitized but realistic Wazuh JSON/log output
3. **Investigation walkthrough** — step-by-step analysis of the relevant fields
4. **Verdict** — true positive or false positive, and why
5. **Detection rule** — the Wazuh rule ID and logic behind the alert

## Getting started

See [00-setup/minimal-setup-for-labs.md](Minimal-Setup-for-Wazuh-Labs.md) for a lightweight Wazuh setup — just enough to generate and view alerts, not a full production deployment guide.

## Resources

- [wazuh-rule-syntax-cheatsheet.md](wazuh-rule-syntax-cheatsheet.md) — quick reference for writing and reading Wazuh detection rules

## About

Part of the [Research Cybersecurity Lab](../) — a collection of hands-on labs and research projects in cybersecurity and AI-driven security tooling.

## Contributing

Suggestions for new lab scenarios are welcome. Open an issue or PR.
