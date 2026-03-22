# Portfolio Projects

Five employer-facing cybersecurity projects built from this lab.
Each project is structured to demonstrate technical depth, documentation ability,
and real-world security reasoning — the three things hiring managers look for.

---

## Project 1 — Detection Engineering with MITRE ATT&CK

**Objective:** Build, test, and document custom detection rules mapped to the MITRE ATT&CK framework.

**What you build:**
- Custom Wazuh rules covering T1110, T1021, T1548, T1136, T1053, T1046
- Atomic Red Team test execution against lab hosts to validate detections
- Detection gap analysis — what fires, what doesn't, and why
- Sigma rule versions of each detection (vendor-neutral format)

**Lab components:** Wazuh (VLAN 40), all lab hosts as targets, Atomic Red Team on RED_TEAM VLAN

**Deliverables:**
- GitHub repo: `blerdmh-detection-engineering`
- `/rules/` — Wazuh XML + Sigma YAML per detection
- `/tests/` — Atomic Red Team commands used to validate each rule
- `/docs/` — Detection rationale, false positive analysis, tuning notes
- `README.md` — ATT&CK navigator layer showing coverage

**Resume bullet:**
> Engineered 6 custom SIEM detection rules mapped to MITRE ATT&CK, validated with Atomic Red Team tests, and documented gap analysis — achieving detection coverage for brute force, lateral movement, privilege escalation, persistence, and reconnaissance techniques.

---

## Project 2 — Active Directory Attack and Defense

**Objective:** Deploy a Windows AD environment, execute the AD kill chain, and build detections for each step.

**What you build:**
- Windows Server 2022 DC + 2 Windows 10 workstations (on PURPLE_LAB VLAN 62)
- Execute: password spray → Kerberoasting → pass-the-hash → DCSync
- Wazuh + Sysmon detections for each technique
- Hardening guide based on what you found

**Lab components:** AMDPVE VMs on VLAN 62, Wazuh on VLAN 40

**Deliverables:**
- GitHub repo: `blerdmh-ad-lab`
- `/attack/` — documented attack commands with screenshots
- `/detections/` — Wazuh rules + Sysmon config for each technique
- `/hardening/` — Group Policy and AD hardening applied
- Incident report written as if it were a real engagement

**Resume bullet:**
> Deployed an Active Directory lab environment, executed a full attack kill chain (Kerberoasting, pass-the-hash, DCSync), built Wazuh/Sysmon detections for each technique, and documented findings in a structured incident report.

---

## Project 3 — Honeypot Threat Intelligence Pipeline

**Objective:** Deploy honeypots, capture real attack data, and automate IOC-based blocking.

**What you build:**
- OpenCanary honeypot on VLAN 50 (IOT_HONEYPOT) — simulates SSH, HTTP, SMB
- Parse OpenCanary logs in Wazuh → extract attacking IPs
- MISP instance for IOC storage and enrichment
- Automation: new IOC in MISP → pfSense alias update → auto-block

**Lab components:** VLAN 50 VM, Wazuh, MISP, pfSense API

**Deliverables:**
- GitHub repo: `blerdmh-threat-intel`
- `/honeypot/` — OpenCanary config + deployment
- `/automation/` — pfSense API scripts for alias updates
- `/analysis/` — 30-day threat intelligence report from captured data
- Attacker TTP analysis — what techniques did real scanners use?

**Resume bullet:**
> Built an automated threat intelligence pipeline: OpenCanary honeypot → Wazuh alert → MISP IOC enrichment → pfSense auto-block, capturing and analyzing real attacker TTPs over 30 days.

---

## Project 4 — IoT Security Assessment

**Objective:** Analyze IoT device behavior, firmware, and network traffic for vulnerabilities.

**Lab components:** VLAN 50 isolated environment, Hak5 Pineapple, ESP32 devices, Pwnagotchi, Wireshark/Zeek

**What you build:**
- Traffic capture of IoT devices communicating to cloud endpoints
- Protocol analysis — what are they sending? Is it encrypted?
- ESP32 firmware extraction and analysis
- Rogue AP assessment with Hak5 Pineapple
- Hardening recommendations based on findings

**Deliverables:**
- GitHub repo: `blerdmh-iot-assessment`
- `/captures/` — Zeek logs and Wireshark captures (sanitized)
- `/firmware/` — ESP32 firmware analysis notes
- `/report/` — Penetration testing style assessment report
- Remediation recommendations with priority ratings

**Resume bullet:**
> Conducted a security assessment of lab IoT devices — capturing and analyzing network traffic, extracting ESP32 firmware for analysis, and producing a structured vulnerability report with remediation recommendations.

---

## Project 5 — SOAR Automation Pipeline

**Objective:** Build an automated security response pipeline that reduces mean time to respond.

**What you build:**
- Shuffle SOAR deployment on VLAN 40
- Workflow 1: Wazuh alert → Shuffle → VirusTotal enrichment → TheHive case → Discord alert
- Workflow 2: SSH brute force detected → auto-block IP in pfSense → notify
- Document mean time to respond before and after automation

**Lab components:** Wazuh (VLAN 40), Shuffle SOAR, TheHive, pfSense API, VirusTotal API

**Deliverables:**
- GitHub repo: `blerdmh-soar`
- `/workflows/` — Shuffle workflow exports (JSON)
- `/docs/` — workflow diagrams, automation logic documentation
- Demo video — screen recording of end-to-end alert → case → block
- Before/after MTTR comparison

**Resume bullet:**
> Designed and implemented a SOAR automation pipeline integrating Wazuh, Shuffle, TheHive, and pfSense — automating IOC enrichment via VirusTotal and SSH brute-force auto-blocking, reducing mean time to respond from 15 minutes to under 90 seconds.

---

## Interview Preparation

For any of these projects, prepare to discuss:

**Tool choices:** Why Wazuh over Splunk? Why Zeek alongside Suricata?
Know one alternative for every tool you chose and why you chose yours.

**What didn't work:** Interviewers value honesty about troubleshooting.
Keep notes on failures and how you resolved them.

**Mapping to enterprise:** "In a production SOC, this would be replaced by X, but the underlying concept is the same."

**Detection gaps:** What did you miss, and why? Gap analysis is as impressive as coverage.

**Scale considerations:** How would this design change at 10,000 endpoints vs 10?
