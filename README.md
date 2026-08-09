# Wazuh IR Lab: Detecting SSH Brute-Force Attacks on a Compromised Endpoint

- **Tool:** Wazuh (open-source SIEM/XDR)
- **Author:** David Matute-Jimenez
- **Course:** BCYB644-Intro-to-Inform-and-Cybersecurity

---

## 1. Tool Overview

Wazuh is a free, open-source security platform that combines SIEM (Security Information and Event Management) and XDR (Extended Detection and Response) capabilities. It collects log data from monitored endpoints, correlates that data against a rule engine, and generates real-time alerts when suspicious or malicious activity is detected.

Wazuh is built around three main components: a lightweight **agent** installed on each monitored endpoint that collects and forwards log data; a central **manager** that analyzes that data against its rule engine and decoders; and an **indexer and dashboard** that store alerts and logs and provide a web interface for investigation.

Wazuh solves several core problems in cybersecurity: it enables near real-time threat detection (such as flagging repeated failed login attempts as a possible brute-force attack), gives analysts centralized visibility across an environment instead of manually checking individual system logs, maps alerts to MITRE ATT&CK techniques for deeper context, and supports custom detection rules for threats the default ruleset doesn't cover.

In the incident response lifecycle, Wazuh is most directly tied to the **Identification** phase, since it turns raw log data into an actionable alert an analyst can investigate. This project demonstrates that role directly: Wazuh detects an SSH brute-force attack against a simulated victim endpoint, generates an alert, and supports the follow-up analysis (custom rule creation, MITRE mapping) that comes after initial detection.

---

## 2. Tool Requirements, Setup & Workflow

### VM Setup
**Author: David**

## To Run This Project (Step-By-Step Guide)

### 2.1 Download and Install UTM

**Step 1:** Visit https://mac.getutm.app/, download UTM for macOS, open the `.dmg`, and drag UTM into the Applications folder. Launch it and grant any permissions prompted.

---

### 2.2 Download Ubuntu Server 24.04.4 (ARM64)

**Step 2:** Visit https://ubuntu.com/download/server and select the **64-bit ARM (AArch64) server install image**, since UTM on Apple Silicon requires ARM64, not the standard x86 ISO. Download Ubuntu Server **24.04.4 LTS** (Noble Numbat).

---

### 2.3 Deploy Ubuntu Server as VM1 (Wazuh Manager)

**Step 3:** In UTM, click **Create a New Virtual Machine** and select the Ubuntu Server ARM64 ISO as the boot image.

**Step 4:** Allocate hardware resources. Due to the 8GB RAM constraint on the host machine, this VM was configured with 5GB RAM, 3 CPU cores, and 30GB storage.

![VM1 Hardware Configuration](ProjectImages/utm-vm-hardware-config.png)

**Step 5:** Name the VM **VM1-Wazuh-Manager** and complete creation.

---

### 2.4 Install Ubuntu Server on VM1

**Step 6:** Boot the VM and select **Ubuntu Server** (not the minimized variant) as the installation type. Configure networking (DHCP auto-assigned IP `192.168.64.4`), guided storage using the entire disk, and the user profile (username: `dmatute`).

**Step 7:** In SSH configuration, check **Install OpenSSH server** and leave **Allow password authentication over SSH** enabled, since this is required later for the Hydra brute-force demo.

![SSH Configuration](ProjectImages/ssh-configuration-openssh.png)

**Step 8:** Complete installation, remove the installation medium, and reboot.

> **Troubleshooting note:** The VM hit a GRUB reboot loop on first restart because the ISO wasn't ejected. Fixed by force-stopping the VM in UTM and manually clearing the CD/DVD drive in VM Settings before rebooting.

![GRUB Reboot Loop Issue](ProjectImages/grub-reboot-loop-issue.png)

**Step 9:** VM1 boots successfully into the installed system.

---

### 2.5 Deploy Kali Linux on UTM

Kali Linux was deployed as the attacker VM following Kali's official documentation. Since Kali provides a well-maintained, standard installation process (unlike the ARM64-specific issues encountered with Ubuntu Server), this section is intentionally brief; the official guides cover the process more thoroughly than a re-documentation here would.

**Step 10:** Follow Kali Linux's official UTM installation guide: https://www.kali.org/docs/virtualization/install-utm-guest-vm/

This guide will walk you through downloading the Kali Linux ARM64 image for UTM, creating a new virtual machine, allocating CPU/RAM/storage resources, and importing the Kali Linux image.

### 2.6 Setup and Configure Kali Linux

**Step 11:** Follow Kali Linux's installation and setup guide: https://www.kali.org/docs/installation/hard-disk-install/

This guide will walk you through initial boot and configuration, partitioning and disk setup, user account creation, system updates, and desktop environment setup.

### 2.7 Login to Kali Linux

**Step 12:** At the Kali Linux login screen, log in with username `kali` / password `kali`.

**Step 13:** Confirm the VM is networked correctly by checking its assigned IP address (`192.168.64.3`).

![Kali IP Address Check](ProjectImages/kali-ip-address-check.png)

---

### 2.8 Confirm Network Connectivity Between VMs

**Step 14:** With both VMs running in UTM's Shared Network mode, ping VM1 from Kali to confirm connectivity.

![Ping Kali to VM1 Success](ProjectImages/ping-kali-to-vm1-success.png)

**Step 15:** Confirm SSH access from the host Mac terminal into VM1.

![SSH Login Success VM1](ProjectImages/ssh-login-success-vm1.png)

---

### 2.9 Wazuh Installation on VM1

**Step 16:** With VM1 up and networked, the default Wazuh install script was downloaded and run:

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh && sudo bash wazuh-install.sh -a
```

The script immediately rejected the system, even though VM1 is genuinely 64-bit, just ARM64, not x86_64:

```
ERROR: Uncompatible system. This script must be run on a 64-bit system.
```

![Wazuh Install Architecture Error](ProjectImages/wazuh-install-arch-error.png)

**Fix:** Wazuh's 4.14 branch includes proper ARM64 support. Switching to the version-specific installer resolved the issue and the install proceeded normally:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

![Wazuh 4.14 Install Starts](ProjectImages/wazuh-414-install-starts.png)

**Step 17:** Partway through, the installer failed at the **Wazuh dashboard** stage and automatically rolled back the entire installation. Investigating the install log revealed the real cause was a **disk-full error**: `df -h /` showed the root partition at only 14G total with 58% used, far less than the 30GB the VM was allocated. Ubuntu's guided LVM partitioning had under-allocated the volume group, leaving space unclaimed.

![df -h Showing 58% Used](ProjectImages/df-h-showing-58-percent-used.png)

**Fix:** Extended the logical volume to claim the unallocated space:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

**Step 18:** Re-ran the installer. This time it completed cleanly through every stage, including the dashboard, and printed the final credentials:

```
User: admin
Password: ngPtuRAi?RIMgRaUEPZKHcaf9yT4XBF7
```

![Wazuh Install Success, Credentials](ProjectImages/wazuh-install-success-credentials.png)

**Step 19:** Logging into the dashboard initially returned a `TimeoutException` retrieving internal user configuration from the indexer, an HTTP 500 error.

![Dashboard Timeout Error, Indexer](ProjectImages/dashboard-timeout-error-indexer.png)

Diagnosis traced this back to the disk filling up a second time. The indexer's own index and log growth had consumed the remaining space, and the earlier `lvextend` fix had only reclaimed space *within* the existing 30GB virtual disk, leaving nothing left to extend into.

**Fix:** Resized the actual UTM virtual disk (30GB to 40GB) in VM Settings, then claimed the new space inside the guest OS with `growpart`, `pvresize`, `lvextend`, and `resize2fs`, and restarted all three Wazuh services in order (indexer, then manager, then dashboard).

**Step 20:** With the disk properly sized, the dashboard loaded successfully with no further errors.

This closed out the Wazuh installation. VM1 was now running a stable indexer, manager, and dashboard stack with adequate disk headroom, setting up the self-monitoring configuration described in Section 2.10.

---

### 2.10 Architecture Note: VM1's Dual Role

**Note:** Due to hardware constraints, VM1 serves as both the Wazuh manager and the monitored "victim" endpoint. This is possible because every Wazuh manager includes a built-in local agent (reserved ID `000`) that self-monitors the host it runs on; no separate agent installation is required. In a production deployment, the manager and monitored endpoints would typically run on separate systems, with agents deployed independently to each monitored host.

VM1 (`192.168.64.4`) therefore acts as both the infrastructure running Wazuh (manager, indexer, dashboard) *and* the simulated victim endpoint, the "compromised employee laptop" targeted by the SSH brute-force attack demonstrated in Section 4. See Section 3 for the `agent_control -l` output confirming agent 000's active status.

---

### 2.11 Workflow Diagram

The diagram below shows Wazuh's general operational pipeline: an agent collects raw log data, the manager decodes and evaluates it against the rule engine, matched events become alerts stored in the indexer, and an analyst reviews them through the dashboard. To make this concrete, the diagram is labeled with the actual rule IDs generated during our SSH brute-force scenario, showing how a single event can resolve to either a routine alert (Rule 5715, level 3) or, once correlated with our custom rule, a high-severity alert (Rule 100100, level 12). The full incident narrative behind these rule IDs is covered in Section 4.

![Wazuh Workflow Diagram](ProjectImages/wazuh_workflow_horizontal.png)

---

## 3. Core Features

### 3.1 Architecture: Agents, Manager, Indexer, and Dashboard

Wazuh is built around four architectural components that work together to move raw data into actionable intelligence.

**Agents** are lightweight programs installed on monitored endpoints (servers, workstations, cloud instances) that collect log data, monitor file integrity, and report system state back to a central manager. Agents run with minimal system overhead and are each identified by a unique agent ID. Every Wazuh manager also includes a built-in local agent (reserved ID `000`) that self-monitors the host it runs on, without requiring separate installation.

The **manager** is the analysis engine, and functions in two stages. A **decoder** first parses raw, unstructured log text into structured fields, for example, taking a single line of SSH log output and extracting the source IP, destination username, and port as separate, queryable fields. This matters because raw text alone can't be reliably filtered, aggregated, or correlated; structuring it is what makes detection possible in the first place. Once decoded, the **rule engine** evaluates those structured fields against Wazuh's ruleset: thousands of XML-defined rules, each carrying a severity level (0 to 15, with higher numbers indicating greater urgency), that determine whether and how loudly an event should generate an alert. Rules can match single events outright, or correlate patterns across multiple events over a defined time window, for instance, treating one failed login as routine but a sudden burst of them from the same source as evidence of an attack. This correlation capability is what separates a SIEM's rule engine from simple log storage: it lets the same underlying event data support both "did this happen" and "does this look like an attack pattern" simultaneously.

The **indexer**, built on OpenSearch, stores and indexes every alert the manager generates, making the full history of alerts and logs searchable at scale. This is the component that makes months of accumulated security data usable rather than just archived.

The **dashboard** is a web-based interface that queries the indexer and displays results. It does not generate or store any data itself; it is purely a visualization and investigation layer for analysts, organized into a home page (shown below) and several purpose-built modules covering each of Wazuh's feature areas.

![Wazuh Dashboard Overview](ProjectImages/wazuh-dashboard-overview-success.png)

### 3.2 Endpoint Security

This category covers detection and hardening at the individual host level, focused on the state of a system rather than network traffic passing through it. **Configuration Assessment** scans a system's settings against known security baselines (such as CIS benchmarks) to flag misconfigurations that could weaken its security posture even in the absence of an active attack. **Malware Detection** checks for indicators of compromise associated with known malware signatures or cyberattack patterns, correlating local system state against threat intelligence feeds. **File Integrity Monitoring (FIM)** tracks changes to files and directories, including permissions, ownership, content, and timestamps, and alerts when unauthorized or unexpected modifications occur. FIM is particularly valuable for detecting persistence mechanisms or tampering that happens *after* an initial compromise, since an attacker who has already gained access will often modify configuration files, add backdoors, or alter system binaries as a next step.

### 3.3 Threat Intelligence

This category focuses on understanding attacker behavior across collected data, rather than flagging individual events in isolation. **Threat Hunting** provides a searchable, aggregated view of security alerts, letting an analyst browse alert volume, top triggered rules, and affected hosts over a given time window, rather than reviewing raw logs line by line. This shifts investigation from reactive ("what does this one alert mean") to proactive ("what patterns exist across all my alerts right now"). **Vulnerability Detection** cross-references installed software versions against known vulnerability databases (such as CVE feeds) to flag applications with unpatched, publicly known weaknesses, useful for identifying exposure *before* it's exploited, rather than only detecting exploitation after the fact. The **MITRE ATT&CK** module maps detected alerts directly to specific adversary tactics and techniques from the MITRE ATT&CK framework, a widely used industry-standard taxonomy of attacker behavior maintained by MITRE Corporation. This mapping happens automatically for rules that correlate to known techniques, meaning an alert isn't just "something bad happened" but is tied to a specific, named point in an attacker's likely playbook (for example, "Brute Force" under the "Credential Access" tactic), giving analysts a standardized reference point instead of relying solely on their own interpretation of severity or intent.

### 3.4 Security Operations

This category centers on compliance and organizational hygiene rather than direct threat detection. **IT Hygiene** assesses systems, software, and network configuration at scale for misconfigurations and anomalies, functioning as a broader housekeeping check across an environment. The remaining tools, **PCI DSS**, **GDPR**, **HIPAA**, **NIST 800-53**, and **TSC**, each automatically map triggered alerts to the specific controls or clauses of their respective regulatory or industry framework. This is a meaningful time-saver in real organizational use: rather than an analyst or compliance officer manually cross-referencing every alert against, say, PCI DSS requirement 10.2.4, Wazuh attaches that mapping to the alert itself at the moment it's generated, as part of the rule's own metadata.

### 3.5 Cloud Security

This category extends Wazuh's monitoring beyond on-premises endpoints into cloud and SaaS environments, reflecting the reality that most modern organizations run infrastructure and services outside a traditional network perimeter. Dedicated integrations exist for **Docker** (monitoring container lifecycle events such as creation, starting, and stopping), **Amazon Web Services** and **Google Cloud Platform** (security events pulled directly via each provider's respective API), **GitHub** (audit log monitoring for organizational activity), and **Office 365** / **Microsoft Graph API** (Microsoft 365 service security events). These integrations funnel cloud-native security events through the same manager/indexer/dashboard pipeline used for on-premises endpoints, meaning an organization doesn't need a separate tool or dashboard to correlate on-prem and cloud activity together.

### 3.6 `wazuh-logtest`: Offline Rule Validation

Wazuh includes a command-line utility, `wazuh-logtest`, that allows an analyst to test how a given log line would be decoded and which rules it would match, entirely offline, without needing to generate a live event or query the indexer/dashboard. Each submitted log line runs through the full decode-then-match pipeline, and the tool reports exactly which rule ID and severity level fired, along with any associated MITRE ATT&CK mapping and firing count if the rule is frequency-based. This makes it a genuinely practical tool in two distinct scenarios: troubleshooting why an *existing* rule isn't firing as expected on production data, and validating a *newly written* custom rule's logic before deploying it against live traffic, where a mistake could mean either missed detections or a flood of false positives.

```bash
sudo /var/ossec/bin/wazuh-logtest
```

---

## 4. Practical Application

### 4.1 Scenario

This project simulates a common insider-risk scenario: a disgruntled employee's company laptop, VM1 in this environment, is left with weak SSH credentials, no account lockout policy, and password authentication enabled. An attacker (Kali Linux, VM2, `192.168.64.3`) discovers the exposed SSH service and runs a brute-force attack against it using Hydra. The victim endpoint (VM1, `192.168.64.4`), running the Wazuh manager with its built-in local agent (`000`, see Section 2.10), detects and logs the attack in real time.

The `agent_control -l` output below confirms agent `000` (`wazuh-manager`, server role) was active and reporting locally at the time of the attack, exactly the dual-role setup described in Section 2.10.

![agent_control Confirms Agent 000 Active](ProjectImages/agent-control-confirms-agent-000.png)

The goal of this exercise was twofold: first, confirm that Wazuh's out-of-the-box rules detect the brute-force attempts and the eventual successful login as two separate events; second, identify and close the gap between those two default alerts with a custom correlation rule, since a successful login immediately following a string of failures is a materially more urgent event than either one alone.

### 4.2 Executing the Attack

From Kali, Hydra was run against VM1's SSH service using a short custom wordlist of common weak passwords, simulating an attacker guessing repeatedly against a real, exposed SSH login:

```bash
hydra -l dmatute -P wordlist.txt -t 4 ssh://192.168.64.4
```

`wordlist.txt` contained eleven common weak passwords (`123456`, `password`, `letmein`, `qwerty`, `welcome1`, `changeme`, `admin123`, `default`, `P@ssw0rd`, `guest`, `admin`). Hydra worked through the list with four concurrent tasks and found a valid match: username `dmatute`, password `admin`, completing at `2026-08-08 12:50:41`.

![Hydra Brute Force Attack Output](ProjectImages/hydra-attack-successful.png)

### 4.3 Default Wazuh Detection: The Severity Gap

Each Hydra attempt generated a corresponding `sshd` authentication log entry on VM1 (visible directly in `/var/log/auth.log`), which the local agent (`000`) forwarded to the manager for decoding and rule evaluation.

![auth.log Showing Failed and Accepted SSH Attempts](ProjectImages/auth-log-tail-failed-and-accepted.png)

Three default Wazuh rules fired over the course of the attack, visible together in the Threat Hunting **Events** view:

- **Rule `5760`** (level 5) fired on each individual failed SSH password attempt, the baseline "authentication failed" rule.
- **Rule `5763`** (level 10) fired once eight failed attempts from the same source IP (`192.168.64.3`) crossed Wazuh's built-in frequency threshold, escalating from isolated failures to a recognized brute-force pattern: *"sshd: brute force trying to get access to the system. Authentication failed."*
- **Rule `5715`** (level 3) fired on the final, successful SSH login, *"sshd: authentication success"*, the same rule that would fire for any ordinary, legitimate login.

![Threat Hunting Events Showing Rules 5763 and 5715](ProjectImages/threat-hunting-events-5763-5715.png)

Expanding rule `5763`'s full alert record confirms the mechanics behind the escalation: `rule.frequency` shows `8`, meaning eight failed attempts within the detection window were required to trigger it, and `previous_output` lists the individual failed-login log lines that were rolled up into the single brute-force alert. The alert is also pre-mapped to **T1110 (Brute Force)** under the Credential Access tactic, and to GDPR, HIPAA, PCI DSS, NIST 800-53, and TSC compliance controls, all automatically, as part of the rule's own metadata (see Section 3.4).

![Rule 5763 Full Alert Detail](ProjectImages/rule-5763-alert-detail.png)

Expanding the successful-login alert shows rule `5715` firing at level 3, the same severity level as any routine login, despite following immediately after the level-10 brute-force alert above. It carries its own MITRE mapping (**T1078 – Valid Accounts**, **T1021 – Remote Services**) reflecting that a login *did* succeed, but nothing in this record alone indicates the login was connected to an attack.

![Rule 5715 Full Alert Detail](ProjectImages/rule-5715-alert-detail.png)

This is where the default ruleset's real limitation shows up: rule `5715` treats a successful login exactly the same whether it's a routine morning sign-in or the tail end of a brute-force attack that fired `5763` seconds earlier. Nothing in the default ruleset connects the two events. An analyst scanning a dashboard full of alerts could see the level-10 brute-force alert, see a separate level-3 login-success alert a moment later, and have no built-in signal tying them together as a single, likely-successful compromise. This severity gap, a genuine security-relevant successful login sitting at the same low severity as any other login, was the specific problem the custom rule in Section 4.4 was written to solve.

For a baseline reference point, the Threat Hunting dashboard *before* the attack traffic is included below, showing routine activity only: 7 total events, 3 authentication failures, 2 authentication successes, and zero level-12-or-above alerts.

![Threat Hunting Dashboard Before the Attack](ProjectImages/threat-hunting-dashboard-before.png)

### 4.4 Building the Correlation Rule

To close this gap, a custom Wazuh correlation rule was written in `/var/ossec/etc/rules/local_rules.xml` on VM1. The goal was straightforward: if a successful login (`5715`) is immediately preceded by a brute-force alert (`5763`) from the same source IP, treat that as a high-severity event, since it strongly suggests the login succeeded *because* of the preceding brute-force attempt, not in spite of it.

The first version of the rule used the wrong tag to reference the brute-force alert:

```xml
<id>^5715$</id>
```

This produced no match during testing. The bug was a conceptual mix-up rather than a typo: `<id>` matches a numeric value *extracted from within a log's fields*, not the ID of a rule that has already fired. Since rule `5763`'s ID isn't a field inside the log text itself, it can never be matched this way. What was actually needed was `<if_sid>`, which tells the rule engine "only evaluate this rule if the rule with this ID already matched this event," and `<if_matched_sid>`, which checks whether a *different, prior* rule fired recently for the same agent. Correcting the tags produced the working rule:

```xml
<rule id="100100" level="12">
  <if_matched_sid>5763</if_matched_sid>
  <if_sid>5715</if_sid>
  <same_source_ip />
  <description>Successful SSH login from $(srcip) following brute force attempts - possible account compromise</description>
  <mitre>
    <id>T1110</id>
    <id>T1078</id>
  </mitre>
  <group>authentication_success,brute_force_success,</group>
</rule>
```

In plain terms: rule `100100` only evaluates on events that already matched rule `5715` (a successful login), and only fires if rule `5763` (brute-force detected) had *also* matched recently for the same source IP. The `<same_source_ip />` tag ensures the correlation only applies when both the failed attempts and the successful login came from the identical attacking IP, not just any brute-force alert and any login happening to occur near each other in time. When both conditions hold, the rule fires at level 12, a significant jump from `5715`'s level 3, and attaches two MITRE ATT&CK techniques: **T1110** (Brute Force) for the attack method, and **T1078** (Valid Accounts) for the fact that the attacker is now operating with a legitimate account's credentials, which is arguably the more dangerous half of the story, since a valid-account login is far harder to distinguish from normal user activity going forward.

### 4.5 Validating the Rule with `wazuh-logtest`

Before trusting rule `100100` against live traffic, it was validated offline using `wazuh-logtest` (introduced in Section 3.6), which allows a rule's logic to be tested against real captured log lines without needing to re-run the attack live.

The real `full_log` lines captured from the actual Hydra attack were fed into `wazuh-logtest` in sequence: the failed-login lines first, to build up past Wazuh's internal frequency threshold and trigger rule `5763`, followed by the single successful-login line. On the successful-login line, `wazuh-logtest` confirmed rule `100100` fired exactly as designed:

```
**Phase 3: Completed filtering (rules).**
        id: '100100'
        level: '12'
        description: 'Successful SSH login from 192.168.64.3 following brute force attempts - possible account compromise'
        groups: '['local', 'syslog', 'sshd', 'authentication_success', 'brute_force_success']'
        firedtimes: '1'
        frequency: '2'
        mail: 'True'
        mitre.id: '['T1110', 'T1078']'
        mitre.tactic: '['Credential Access', 'Defense Evasion', 'Persistence', 'Privilege Escalation', 'Initial Access']'
        mitre.technique: '['Brute Force', 'Valid Accounts']'
**Alert to be generated.
```

![wazuh-logtest Output Rule 100100 Firing](ProjectImages/wazuh-logtest-rule-100100-firing.png)

The output confirms all of the rule's design goals in one pass: the correct rule ID and elevated level fired (`100100`, level 12, versus `5715`'s level 3), the description correctly rendered the attacking source IP into the message, `**Alert to be generated` confirms it would actually produce a dashboard alert (not just a silent rule match), and both intended MITRE ATT&CK techniques (T1110, T1078) were attached automatically, expanding out to five MITRE tactics in total.

This same behavior is confirmed a second time by the live alert itself once the rule was deployed and the attack re-triggered it for real. Rule `100100` appears directly in the Threat Hunting **Events** list alongside the routine `5760`/`5715` traffic around it, exactly where an analyst would encounter it in practice:

![Rule 100100 Appearing in the Events List](ProjectImages/rule-100100-in-events-list.png)

And the full alert record confirms every field matches what `wazuh-logtest` predicted, now with the actual timestamp, agent, and source IP attached: `rule.id` `100100`, `rule.level` `12`, `rule.groups` including `brute_force_success`, and `rule.mitre.id` `T1110, T1078`.

![Rule 100100 Full Alert Detail](ProjectImages/rule-100100-alert-detail.png)

### 4.6 Before / After: Dashboard Impact

Comparing the Threat Hunting dashboard before (Section 4.3) and after the attack makes the detection chain's effect on the overall alert picture concrete. After the attack, total events for the window rose from 7 to **26**, authentication failures rose from 3 to **19** (the Hydra password-guessing attempts), and authentication successes rose from 2 to **4**.

![Threat Hunting Dashboard After the Attack](ProjectImages/threat-hunting-dashboard-after.png)

The **Top 10 MITRE ATT&CKS** panel breaks this down by technique: **Password Guessing** (16 occurrences) and **SSH** (11) dominate, reflecting the volume of individual failed Hydra attempts, while **Valid Accounts** (4), **Brute Force** (3), and **Remote Services** (2) mark the smaller number of alerts, including rule `100100`, tied to the attack actually succeeding.

![Top 10 MITRE ATT&CK Techniques Table](ProjectImages/top-10-mitre-attacks-table.png)

Live re-running of the Hydra attack against the dashboard end-to-end was also captured directly (rather than relying on `wazuh-logtest` alone): the events list, the individual `5763` and `5715` alert records, and the `100100` alert record above are all pulled from that live run, so this project has both offline rule-logic validation *and* live-traffic proof that the full pipeline, from Hydra's first failed attempt through the custom rule firing, works end to end.

### 4.7 Summary: MITRE ATT&CK Mapping

| Rule ID | Level | Trigger | MITRE Technique(s) |
|---|---|---|---|
| `5760` | 5 | Single failed SSH login attempt | — |
| `5763` | 10 | 8 failed attempts from same source IP cross frequency threshold | T1110 – Brute Force |
| `5715` | 3 | Any successful SSH login (default, no context) | T1078 – Valid Accounts, T1021 – Remote Services |
| `100100` (custom) | 12 | Successful login (`5715`) immediately preceded by brute force (`5763`) from same source IP | T1110 – Brute Force, T1078 – Valid Accounts |

The end-to-end result is a detection chain that mirrors how a real analyst would reason about this incident: individual failures are noise, a burst of failures is a brute-force attempt worth flagging, and a login that succeeds right after that burst isn't just "a login," it's a likely account compromise that deserves a level-12 alert and an explicit MITRE ATT&CK trail, rather than getting lost at the same severity as a routine sign-in.

---

## 5. Strengths & Limitations

*(TBD, Strengths: real-time detection, customizable rules, MITRE integration. Limitations/scope tradeoffs: no active-response demo, no FIM demo, RAM constraint rationale, including VM1's dual manager/endpoint role (see Section 2.10). Lessons learned: ARM64 install issue, disk sizing x2, GRUB reboot loop.)*

---

## 6. References

*(TBD, Wazuh docs, MITRE ATT&CK, UTM/Ubuntu/Kali install guides used)*
