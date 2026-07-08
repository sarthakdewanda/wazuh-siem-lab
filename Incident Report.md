# Incident Report Analysis

Structured using the NIST Cybersecurity Framework (CSF) 2.0: Summary followed by the six core functions, Govern, Identify, Protect, Detect, Respond, Recover.

| | |
|---|---|
| **Report ID** | IR-2026-0708-001 |
| **Date** | 8 July 2026 |
| **Analyst** | _<Sarthak Dewanda>_ |
| **Environment** | Wazuh SIEM home lab (VirtualBox) |
| **Exercise type** | Purple-team detection validation (controlled simulation) |

> **Scope note:** This documents a controlled home-lab exercise. The adversary activity was generated intentionally with Atomic Red Team to validate detection and response capability. No real intrusion occurred. It is written in the CSF report format to reflect how the detection would be handled operationally.

---

## Summary

On 8 July 2026, the Wazuh SIEM detected **System Owner/User Discovery** activity (MITRE ATT&CK **T1033**) on the Windows 11 endpoint **win-lab (agent 004)**. The activity, execution of `whoami.exe`, was caught by a custom detection rule (**rule ID 100010**, severity level 10) built on Sysmon process-creation telemetry.

During investigation, the same host was found to carry **298 known vulnerabilities** in its software inventory, including multiple Critical, network-exploitable CVEs (for example CVE-2026-47291). There is no evidence the vulnerabilities were exploited, but reconnaissance activity on a critically vulnerable host is an elevated-priority signal and is treated as such below.

---

## Govern

The risk management strategy and expectations that guided this exercise.

- This activity was authorized and scoped to a personal home lab environment. Atomic Red Team execution was limited to a single named technique (T1033) run against a dedicated, snapshotted VM, not the physical host.
- A VirtualBox snapshot (`clean-before-ART`) was taken prior to detonation as the rollback control, defining the acceptable risk boundary before any test ran.
- The Microsoft Defender exclusion added for the Atomic Red Team folder was scoped narrowly to `C:\AtomicRedTeam` and treated as a temporary lab exception rather than a standing policy.
- This report is itself a governance artifact. It records what was tested, on which asset, under what authorization, and with what outcome, satisfying the expectation that security-relevant activity is documented and reviewable.

---

## Identify

Understand the assets involved and the security risks exposed.

**Assets in scope:**
- **win-lab (agent 004):** Windows 11 Home, build 10.0.26200.8037. Monitored endpoint, the affected asset.
- **Ubuntu Wazuh Manager:** SIEM performing collection, analysis, and correlation.

**Risks identified:**
- The endpoint carries **298 known CVEs** in installed software, spanning Critical, High, and Medium severity.
- Anchor risk: **CVE-2026-47291** (Critical), an integer-overflow flaw with a network attack vector, low complexity, no privileges and no user interaction required (CVSS AV:N/AC:L/PR:N/UI:N, C/I/A all High). An official fix is available.
- Additional Critical exposure includes **CVE-2026-45657** (use-after-free, Windows Kernel).
- Behavioral risk: **discovery activity (T1033)** enumerates the current user context, a common early step attackers take to plan privilege escalation or lateral movement.

The core gap is an internet-class, unpatched vulnerability set on a host that is also showing reconnaissance behavior.

---

## Protect

Safeguards that reduce the likelihood and impact of this activity.

- **Patch management:** apply the available official fixes for CVE-2026-47291, CVE-2026-45657, and the remaining Critical/High findings, prioritizing network-reachable, no-interaction CVEs.
- **Least privilege:** ensure endpoint accounts (for example `vboxuser`) run without unnecessary admin rights, limiting what discovery and follow-on actions can achieve.
- **Application control / allowlisting:** restrict or monitor execution of built-in discovery utilities where feasible.
- **Endpoint hardening:** keep Microsoft Defender enabled and current; the lab exclusion added for testing should be scoped tightly and removed outside exercises.
- **Baselining:** document expected, legitimate administrative use of `whoami` and similar tools so routine activity is distinguishable from suspicious execution.

---

## Detect

How the activity was detected, and how detection can improve.

**How it was detected:**
- **Sysmon** logged the process creation as Event ID 1 on host 004 and forwarded it via the Wazuh agent.
- **Custom Wazuh rule 100010** matched the process image against `(?i)\whoami\.exe` and raised a level-10 alert: "Discovery: whoami.exe executed, possible System Owner/User Discovery (T1033)."
- The rule's `<mitre>` tag mapped the alert to **T1033 (Tactic: Discovery)**, populating the dashboard's MITRE ATT&CK module.
- Wazuh's built-in Sysmon ruleset independently flagged the discovery/PowerShell activity, corroborating the custom detection.

**Detection improvements:**
- Add correlation so discovery alerts on hosts with open Critical CVEs are automatically escalated (the manual pivot performed here is the automation candidate).
- Tune rule 100010's severity to the environment's baseline to reduce noise while preserving visibility.

---

## Respond

Actions taken to contain, neutralize, and analyze the incident.

- Triaged the alert from rule 100010 and confirmed the affected asset (win-lab / 004), technique (T1033), and executed command (`whoami`).
- Pivoted on the asset to its vulnerability inventory and identified 298 findings, anchoring on the Critical CVE-2026-47291 to assess exposure.
- Correlated the detection with the vulnerability posture, concluding that discovery activity on a critically vulnerable host warrants investigation ahead of similar activity on a hardened asset.
- Verified scope: confirmed no evidence of exploitation, lateral movement, or persistence; in this exercise the activity originated from the sanctioned simulation.
- In a production setting the equivalent response would be to isolate the host if warranted, hunt for follow-on activity, and escalate priority due to the Critical, network-reachable vulnerabilities on the same asset.

---

## Recover

Restore normal operations and strengthen against recurrence.

- **Remediate:** apply patches for the Critical/High CVEs on host 004, closing the exploitable exposure that elevated this finding.
- **Re-scan:** run a fresh Wazuh vulnerability inventory after patching to confirm the CVE count drops, closing the loop.
- **Retain the detection:** keep rule 100010 in production monitoring; add correlation-based escalation for discovery-on-vulnerable-host.
- **Lessons learned:** the exercise validated the full detection-to-response path (telemetry, detection, ATT&CK mapping, vulnerability correlation). Detection capability for T1033 on this endpoint is confirmed operational.

---

## Appendix: Key Artifacts

| Type | Value |
|---|---|
| Affected asset | win-lab (agent 004), Windows 11 Home 10.0.26200.8037 |
| Detection rule | Wazuh rule ID 100010 (level 10, `local_rules.xml`) |
| Telemetry | Sysmon Event ID 1, Process Creation |
| Process | `whoami.exe` |
| ATT&CK technique | T1033, System Owner/User Discovery (Tactic: Discovery) |
| Simulation driver | Atomic Red Team test T1033-1 |
| Anchor vulnerability | CVE-2026-47291 (Critical); also CVE-2026-45657 (Critical) |
| Total vulnerabilities on host | 298 |

Evidence screenshots referenced in the project `screenshots/` directory. Prepared as part of a Wazuh SIEM home-lab purple-team exercise.
