# Wazuh IR Lab: Detecting SSH Brute-Force Attacks on a Compromised Endpoint
 
**Tool:** Wazuh (open-source SIEM/XDR)
**Author:** David Matute-Jimenez
**Course:** BCYB644-Intro-to-Inform-and-Cybersecurity

---
 
## 1. Tool Overview
 
Wazuh is a free, open-source security platform that combines SIEM (Security Information and Event Management) and XDR (Extended Detection and Response) features. Basically, it collects logs from whatever systems you're monitoring, runs those logs against a rule engine, and generates alerts in real time when something looks off.
 
It's made up of three main pieces: an **agent** that sits on each monitored machine and forwards log data, a central **manager** that decodes that data and checks it against rules, and an **indexer/dashboard** combo that stores the alerts and gives you a web interface to actually look at them.
 
What Wazuh is really doing is taking messy raw logs and turning them into something a person can act on: catching repeated failed logins before they become an actual breach, giving a team one place to check instead of ten different systems, tagging alerts with MITRE ATT&CK info so they mean something, and letting you write your own rules for anything the defaults don't catch.
 
That core function, raw logs to actionable alert, puts Wazuh mainly in the **Identification** phase of the incident response lifecycle, which is exactly what this project shows off in Section 4: Wazuh catching an SSH brute-force attack against a simulated victim machine and generating an alert for it. That's not the only phase it touches though. Section 5 gets into how some of the other features covered in Section 3 (Configuration Assessment, active response, File Integrity Monitoring) extend Wazuh into Preparation, Containment, Eradication, and even Recovery and Lessons Learned, even though this project's hands-on demo stuck to Identification.
 
---
 
## 2. Tool Requirements, Setup & Workflow
 
### VM Setup
 
## To Run This Project (Step-By-Step Guide)
 
### 2.1 Download and Install UTM
 
**Step 1:** Go to https://mac.getutm.app/, download UTM for macOS, open the `.dmg`, and drag it into Applications. Launch it and click through any permission prompts.
 
---
 
### 2.2 Download Ubuntu Server 24.04.4 (ARM64)
 
**Step 2:** Head to https://ubuntu.com/download/server and grab the **64-bit ARM (AArch64) server install image**, since UTM on Apple Silicon needs ARM64, the regular x86 ISO won't work. I used Ubuntu Server **24.04.4 LTS** (Noble Numbat).
 
---
 
### 2.3 Deploy Ubuntu Server as VM1 (Wazuh Manager)
 
**Step 3:** In UTM, hit **Create a New Virtual Machine** and point it at the Ubuntu Server ARM64 ISO.
 
**Step 4:** Set your hardware. My host machine only has 8GB of RAM, so I gave this VM 5GB RAM, 3 CPU cores, and 30GB storage.
 
![VM1 Hardware Configuration](ProjectImages/utm-vm-hardware-config.png)
 
**Step 5:** Named it **VM1-Wazuh-Manager** and finished setup.
 
---
 
### 2.4 Install Ubuntu Server on VM1
 
**Step 6:** Boot the VM, pick **Ubuntu Server** (not the minimized version), and go through setup. Networking grabbed `192.168.64.4` via DHCP, guided storage used the whole disk, and I set the username to `dmatute`.
 
**Step 7:** When it hits SSH configuration, check **Install OpenSSH server** and leave **Allow password authentication over SSH** on, since I needed this later for the Hydra demo.
 
![SSH Configuration](ProjectImages/ssh-configuration-openssh.png)
 
**Step 8:** Finish install, pull the installation medium, reboot.
 
> **Ran into an issue here:** hit a GRUB reboot loop on the first restart because I forgot to eject the ISO. Fixed it by force-stopping the VM in UTM and manually clearing the CD/DVD drive in settings before rebooting.
 
![GRUB Reboot Loop Issue](ProjectImages/grub-reboot-loop-issue.png)
 
**Step 9:** VM1 boots fine after that.
 
---
 
### 2.5 Deploy Kali Linux on UTM
 
I set up Kali following their official docs. Kali's install process is well-maintained and doesn't have the ARM64 headaches Ubuntu Server gave me, so I'm not going to re-document the whole thing here. Their guide covers it better than I would.
 
**Step 10:** Follow Kali's official UTM install guide: https://www.kali.org/docs/virtualization/install-utm-guest-vm/
 
Covers downloading the ARM64 image, creating the VM, allocating resources, and importing it.
 
### 2.6 Setup and Configure Kali Linux
 
**Step 11:** Follow Kali's setup guide: https://www.kali.org/docs/installation/hard-disk-install/
 
Covers first boot, partitioning, creating a user, updates, and the desktop environment.
 
### 2.7 Login to Kali Linux
 
**Step 12:** Login screen, username `kali`, password `kali`.
 
**Step 13:** Checked the assigned IP to confirm networking (`192.168.64.3`).
 
![Kali IP Address Check](ProjectImages/kali-ip-address-check.png)
 
---
 
### 2.8 Confirm Network Connectivity Between VMs
 
**Step 14:** Both VMs are on UTM's Shared Network mode, so I pinged VM1 from Kali to make sure they could actually talk to each other.
 
![Ping Kali to VM1 Success](ProjectImages/ping-kali-to-vm1-success.png)
 
**Step 15:** Also confirmed I could SSH into VM1 from my Mac's terminal.
 
![SSH Login Success VM1](ProjectImages/ssh-login-success-vm1.png)
 
---
 
### 2.9 Wazuh Installation on VM1
 
**Step 16:** With VM1 up, I ran the standard Wazuh install script:
 
```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh && sudo bash wazuh-install.sh -a
```
 
It rejected the system right away, even though VM1 is genuinely 64-bit, just ARM64, not x86_64:
 
```
ERROR: Uncompatible system. This script must be run on a 64-bit system.
```
 
![Wazuh Install Architecture Error](ProjectImages/wazuh-install-arch-error.png)
 
**Fix:** Turns out Wazuh's 4.14 branch actually supports ARM64 properly. Switched to the version-specific installer and it went through fine:
 
```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```
 
![Wazuh 4.14 Install Starts](ProjectImages/wazuh-414-install-starts.png)
 
**Step 17:** Got partway through and it failed at the dashboard stage, rolled the whole install back. Log pointed to a disk-full error: `df -h /` showed only 14G total with 58% used, way less than the 30GB I gave the VM. Ubuntu's guided LVM partitioning had left most of that space just sitting there unclaimed.
 
![df -h Showing 58% Used](ProjectImages/df-h-showing-58-percent-used.png)
 
**Fix:** Extended the logical volume to grab the unallocated space:
 
```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```
 
**Step 18:** Reran the installer and it went through cleanly this time, dashboard included, and printed the final credentials:
 
```
User: admin
Password: ngPtuRAi?RIMgRaUEPZKHcaf9yT4XBF7
```
 
![Wazuh Install Success, Credentials](ProjectImages/wazuh-install-success-credentials.png)
 
**Step 19:** Tried logging into the dashboard and got a `TimeoutException` pulling internal user config from the indexer, an HTTP 500.
 
![Dashboard Timeout Error, Indexer](ProjectImages/dashboard-timeout-error-indexer.png)
 
Same root problem as before, just round two: the indexer's own index and log growth had already eaten up the space I'd just freed. My earlier fix only cleared space *inside* the existing 30GB disk, there was nothing left to extend into.
 
**Fix:** Actually resized the UTM virtual disk itself (30GB to 40GB) in VM settings, then claimed that new space inside the guest OS with `growpart`, `pvresize`, `lvextend`, and `resize2fs`, and restarted the three Wazuh services in order: indexer, manager, dashboard.
 
**Step 20:** Disk sized properly this time, dashboard loaded with no errors.
 
That closed out the install. VM1 was now running a stable indexer/manager/dashboard stack with actual room to breathe, which set up the self-monitoring setup I get into in Section 2.10.
 
---
 
### 2.10 Architecture Note: VM1's Dual Role
 
**Note:** Because of the RAM constraints on my host machine, VM1 ended up playing double duty. It's both the Wazuh manager and the "victim" endpoint being monitored. This works because every Wazuh manager comes with a built-in local agent (reserved ID `000`) that watches the host it's running on, so I didn't need a separate agent install. In an actual production setup, the manager and the endpoints it watches would normally be separate machines, each running their own agent.
 
So VM1 (`192.168.64.4`) is doing two jobs at once: it's the infrastructure running Wazuh (manager, indexer, dashboard), and it's the simulated victim, the "compromised employee laptop" that gets hit with the SSH brute-force attack in Section 4. Section 3 has the `agent_control -l` output confirming agent 000 was active.
 
---
 
### 2.11 Workflow Diagram
 
This diagram lays out Wazuh's general pipeline: an agent collects raw logs, the manager decodes and evaluates them against the rule engine, matched events become alerts stored in the indexer, and an analyst pulls them up in the dashboard. I labeled it with the actual rule IDs from this project's attack scenario, showing how one event can end up as either a routine alert (Rule 5715, level 3) or, once it's correlated with my custom rule, a high-severity one (Rule 100100, level 12). Section 4 has the full story behind these rule IDs.
 
![Wazuh Workflow Diagram](ProjectImages/wazuh_workflow_horizontal.png)
 
---
 
## 3. Core Features
 
### 3.1 Architecture: Agents, Manager, Indexer, and Dashboard
 
Wazuh has four main pieces working together to turn raw data into something useful.
 
**Agents** are lightweight programs you install on whatever you're monitoring (servers, workstations, cloud instances) that collect logs, watch file integrity, and report system state back to the manager. They're low-overhead and each gets its own agent ID. Every manager also comes with a built-in local agent (ID `000`) that monitors the host it's on, so no extra install needed there.
 
The **manager** is where the actual analysis happens, in two stages. First, a **decoder** takes raw, messy log text and pulls structured fields out of it, for example, taking one line of SSH log output and pulling out the source IP, the username, the port, as separate fields you can actually query. This matters because you can't filter or correlate raw text reliably; structuring it is what makes detection possible at all. Once it's decoded, the **rule engine** checks those fields against Wazuh's ruleset: thousands of XML rules, each with a severity level from 0-15. Rules can match on a single event, or they can correlate multiple events over a time window, so one failed login is treated as normal but a burst of them from the same IP gets flagged as an attack. That correlation piece is really what separates a SIEM from just storing logs: the same data can answer both "did this happen" and "does this look like an attack."
 
The **indexer**, built on OpenSearch, stores every alert the manager generates and keeps months of it searchable instead of just sitting in an archive somewhere.
 
The **dashboard** is the web interface that queries the indexer and shows you results. It doesn't generate or store anything itself; it's purely for looking at data, organized into a home page and a bunch of modules for each feature area.
 
![Wazuh Dashboard Overview](ProjectImages/wazuh-dashboard-overview-success.png)
 
### 3.2 Endpoint Security
 
This category is about the state of an individual host, not the traffic moving through it. **Configuration Assessment** checks a system's settings against known baselines (like CIS benchmarks) and flags misconfigurations even if nothing's actively attacking. **Malware Detection** checks for indicators tied to known malware signatures. **File Integrity Monitoring (FIM)** watches for changes to files and directories (permissions, ownership, content, timestamps) and alerts if something unexpected happens. FIM is especially useful for catching stuff that happens *after* someone's already gotten in, since an attacker with access is usually going to mess with config files, drop a backdoor, or alter a binary as their next move.
 
### 3.3 Threat Intelligence
 
This is more about spotting patterns across all your data instead of reacting to one event at a time. **Threat Hunting** gives you a searchable view of alerts (volume, top rules, affected hosts) over whatever time window you want, instead of scrolling raw logs. It flips investigation from "what does this one alert mean" to "what's actually going on across everything right now." **Vulnerability Detection** checks installed software against known vulnerability databases so you can catch exposure before it gets exploited, not just after. The **MITRE ATT&CK** module tags alerts with actual adversary tactics and techniques automatically, for anything that maps to a known technique. So instead of just "something bad happened," an alert is tied to a specific point in an attacker's likely playbook, like "Brute Force" under "Credential Access," which gives analysts a shared reference point instead of everyone guessing at severity on their own.
 
### 3.4 Security Operations
 
This one's more about compliance and general hygiene than direct threat detection. **IT Hygiene** checks systems, software, and network config at scale for misconfigurations. The rest (**PCI DSS**, **GDPR**, **HIPAA**, **NIST 800-53**, **TSC**) each map triggered alerts to that specific framework's controls automatically. This actually saves a lot of time in practice: instead of an analyst manually checking every alert against, say, PCI DSS 10.2.4, Wazuh just attaches that mapping the moment the alert fires.
 
### 3.5 Cloud Security
 
This extends Wazuh past on-prem into cloud/SaaS stuff, since most orgs aren't running everything inside a traditional network anymore. There's integrations for **Docker** (container lifecycle events), **AWS** and **GCP** (security events pulled directly from each provider's API), **GitHub** (audit log monitoring), and **Office 365**/**Microsoft Graph API**. All of it funnels through the same manager/indexer/dashboard pipeline as on-prem stuff, so you're not stuck using a separate tool to correlate cloud and on-prem activity.
 
### 3.6 `wazuh-logtest`: Offline Rule Validation
 
Wazuh has a command-line tool, `wazuh-logtest`, that lets you feed it a log line and see how it'd be decoded and which rules would match, entirely offline, no live event, no dashboard needed. It reports back the rule ID, severity, any MITRE mapping, and firing count if it's a frequency-based rule. This is genuinely useful for two things: figuring out why a rule isn't firing the way you expect on real data, and testing a new custom rule's logic before you throw it at live traffic, where a mistake could mean missed detections or a wave of false positives.
 
```bash
sudo /var/ossec/bin/wazuh-logtest
```
 
---
 
## 4. Practical Application
 
### 4.1 Scenario
 
This project simulates a pretty common insider-risk situation: a disgruntled employee's company laptop (VM1) is sitting there with weak SSH credentials, no account lockout, and password auth enabled. An attacker (Kali, VM2, `192.168.64.3`) finds the exposed SSH service and runs a brute-force attack against it with Hydra. VM1 (`192.168.64.4`), running the Wazuh manager and its built-in local agent (`000`, see Section 2.10), catches and logs the attack as it happens.
 
Below is the `agent_control -l` output confirming agent `000` (`wazuh-manager`, server role) was active and reporting at the time of the attack, matching the dual-role setup from Section 2.10.
 
![agent_control Confirms Agent 000 Active](ProjectImages/agent-control-confirms-agent-000.png)
 
The goal here was two things: confirm Wazuh's default rules actually catch the brute-force attempts and the successful login as two separate events, then find and close the gap between those two alerts with a custom correlation rule, since a successful login right after a bunch of failures is a way bigger deal than either event on its own.
 
### 4.2 Executing the Attack
 
From Kali, I ran Hydra against VM1's SSH service with a short wordlist of common weak passwords:
 
```bash
hydra -l dmatute -P wordlist.txt -t 4 ssh://192.168.64.4
```
 
`wordlist.txt` had eleven common weak passwords (`123456`, `password`, `letmein`, `qwerty`, `welcome1`, `changeme`, `admin123`, `default`, `P@ssw0rd`, `guest`, `admin`). Hydra ran through it with four concurrent tasks and found a match: username `dmatute`, password `admin`, done at `2026-08-08 12:50:41`.
 
![Hydra Brute Force Attack Output](ProjectImages/hydra-attack-successful.png)
 
### 4.3 Default Wazuh Detection: The Severity Gap
 
Every Hydra attempt showed up as an `sshd` log entry on VM1, right in `/var/log/auth.log`, which the local agent forwarded up to the manager for decoding and rule matching.
 
![auth.log Showing Failed and Accepted SSH Attempts](ProjectImages/auth-log-tail-failed-and-accepted.png)
 
Three default rules fired during the attack, all visible in the Threat Hunting Events view:
 
- **Rule `5760`** (level 5): fires on every single failed SSH attempt, the baseline "auth failed" rule.
- **Rule `5763`** (level 10): fired once eight failed attempts from the same IP crossed Wazuh's frequency threshold: *"sshd: brute force trying to get access to the system. Authentication failed."*
- **Rule `5715`** (level 3): fired on the final successful login, *"sshd: authentication success"*, same rule that fires for any normal login, nothing special about it.
![Threat Hunting Events Showing Rules 5763 and 5715](ProjectImages/threat-hunting-events-5763-5715.png)
 
Looking at rule `5763`'s full alert record, you can see the mechanics: `rule.frequency` is `8`, meaning it took eight failed attempts in the window to trigger, and `previous_output` shows the individual failed-login lines that got rolled up into that one alert. It's already tagged with **T1110 (Brute Force)** under Credential Access, plus GDPR, HIPAA, PCI DSS, NIST 800-53, and TSC compliance mappings, all automatic, no work on my end (see Section 3.4).
 
![Rule 5763 Full Alert Detail](ProjectImages/rule-5763-alert-detail.png)
 
The successful-login alert, rule `5715`, fires at level 3, same severity as literally any normal login, even though it came right after that level-10 brute-force alert above it. It has its own MITRE tags (**T1078, Valid Accounts**, **T1021, Remote Services**), which makes sense given a login did succeed, but there's nothing in the record itself connecting it back to the attack.
 
![Rule 5715 Full Alert Detail](ProjectImages/rule-5715-alert-detail.png)
 
This is where the default ruleset really falls short: rule `5715` treats a successful login exactly the same whether it's someone logging in for their morning shift or the tail end of a brute-force attack that just fired a level-10 alert seconds earlier. Nothing connects the two. An analyst looking at a dashboard could see the brute-force alert, then a separate low-severity login-success alert right after it, with zero built-in signal telling them these are actually the same incident. This gap, a genuinely important login sitting at the same severity as any other one, is exactly what the custom rule in Section 4.4 was built to fix.
 
For a baseline, here's the Threat Hunting dashboard *before* the attack, just normal activity: 7 total events, 3 auth failures, 2 auth successes, zero level-12-or-above alerts.
 
![Threat Hunting Dashboard Before the Attack](ProjectImages/threat-hunting-dashboard-before.png)
 
### 4.4 Building the Correlation Rule
 
To fix this, I wrote a custom Wazuh rule in `/var/ossec/etc/rules/local_rules.xml` on VM1. The idea: if a successful login (`5715`) comes right after a brute-force alert (`5763`) from the same source IP, treat that as a high-severity event, since it strongly suggests the login only succeeded *because* of the brute-force attempt.
 
My first attempt used the wrong tag entirely:
 
```xml
<id>^5715$</id>
```
 
No match, no matter what I tested. Turned out this wasn't a typo, it was a conceptual mistake. `<id>` matches a value pulled from *inside* a log's fields, not the ID of a rule that already fired. Since rule `5763`'s ID isn't part of the log text itself, this could never match. What I actually needed was `<if_sid>` (only evaluate this rule if this rule ID already matched the event) and `<if_matched_sid>` (check if a *different* rule fired recently for the same agent). Fixing the tags got me the working rule:
 
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
 
Full rule file's here: [`custom_rule_100100.xml`](https://github.com/DavidMatuteJimenez/BCYB644-Intro-to-Inform-and-Cybersecurity/blob/main/custom_rule_100100.xml).
 
In plain terms: rule `100100` only evaluates on events that already matched `5715` (successful login), and only actually fires if `5763` (brute-force) had also matched recently for the same source IP. The `<same_source_ip />` tag makes sure it's not just any brute-force alert and any login happening near each other in time, it has to be the *same* attacking IP for both. When it fires, it jumps to level 12 (way up from `5715`'s level 3) and attaches both MITRE techniques: **T1110** (Brute Force) for the attack method, and **T1078** (Valid Accounts) since the attacker's now operating with real, valid credentials. That second one is arguably the scarier half, since a valid-account login is a lot harder to tell apart from normal activity going forward.
 
### 4.5 Validating the Rule with `wazuh-logtest`
 
Before trusting `100100` against live traffic, I validated it offline with `wazuh-logtest` (see Section 3.6), which lets you test rule logic against real captured log lines without having to actually rerun the attack.
 
I fed it the real `full_log` lines from the Hydra attack, in order: failed-login lines first to build past the frequency threshold and trigger `5763`, then the successful-login line. On that last line, `wazuh-logtest` confirmed `100100` fired exactly as designed:
 
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
 
Everything checked out here: right rule ID, right elevated level (`100100`, level 12, vs. `5715`'s level 3), the description correctly pulled in the attacking IP, `**Alert to be generated` confirms it'd actually produce a dashboard alert and not just silently match, and both MITRE techniques (T1110, T1078) got attached automatically, expanding out to five tactics total.
 
I confirmed this a second time with a live alert once the rule was deployed and I re-ran the attack for real. Rule `100100` shows up right in the Threat Hunting Events list next to the regular `5760`/`5715` traffic, exactly where an analyst would actually run into it.
 
![Rule 100100 Appearing in the Events List](ProjectImages/rule-100100-in-events-list.png)
 
The full alert record matches everything `wazuh-logtest` predicted, now with the real timestamp, agent, and source IP attached: `rule.id` `100100`, `rule.level` `12`, `rule.groups` includes `brute_force_success`, `rule.mitre.id` is `T1110, T1078`.
 
![Rule 100100 Full Alert Detail](ProjectImages/rule-100100-alert-detail.png)
 
### 4.6 Before / After: Dashboard Impact
 
Comparing the dashboard before (Section 4.3) and after the attack shows what this actually did to the overall alert picture. After the attack, total events went from 7 to **26**, auth failures went from 3 to **19** (the Hydra attempts), and auth successes went from 2 to **4**.
 
![Threat Hunting Dashboard After the Attack](ProjectImages/threat-hunting-dashboard-after.png)
 
The Top 10 MITRE ATT&CKS panel breaks it down: **Password Guessing** (16) and **SSH** (11) dominate, which tracks with the sheer volume of failed Hydra attempts, while **Valid Accounts** (4), **Brute Force** (3), and **Remote Services** (2) are the smaller set tied to the attack actually succeeding, including `100100`.
 
![Top 10 MITRE ATT&CK Techniques Table](ProjectImages/top-10-mitre-attacks-table.png)
 
I also captured the live re-run of the attack directly instead of just relying on `wazuh-logtest`: the events list, the `5763` and `5715` records, and the `100100` record above are all pulled from that live run. So this has both offline validation of the rule logic and live-traffic proof that the whole pipeline actually works end to end, from Hydra's first failed attempt through the custom rule firing.
 
A full recording of this live re-run, from the initial Hydra attack through the `100100` alert firing, is available here: [Wazuh IR Lab, SSH Brute-Force Attack Detection Demo](https://youtu.be/5dhVRPoot-M)
 
### 4.7 Summary: MITRE ATT&CK Mapping
 
| Rule ID | Level | Trigger | MITRE Technique(s) |
|---|---|---|---|
| `5760` | 5 | Single failed SSH login attempt | — |
| `5763` | 10 | 8 failed attempts from same source IP cross frequency threshold | T1110, Brute Force |
| `5715` | 3 | Any successful SSH login (default, no context) | T1078, Valid Accounts; T1021, Remote Services |
| `100100` (custom) | 12 | Successful login (`5715`) immediately preceded by brute force (`5763`) from same source IP | T1110, Brute Force; T1078, Valid Accounts |
 
The end result is a detection chain that basically mirrors how an analyst would actually reason through this: one failure is noise, a burst of failures is worth flagging, and a login that succeeds right after that burst isn't just "a login." It's a likely compromise that deserves a level-12 alert and a MITRE trail, not the same severity as someone logging in for their shift.
 
A level-12 `100100` alert is the kind of thing that should trigger a real response, not just sit in a log. If I were an analyst seeing this fire, I'd want to: force an immediate password reset (treat the account as compromised until proven otherwise), check `dmatute`'s login history and command activity for anything past the initial SSH session, and block or rate-limit `192.168.64.3` at the firewall while investigating. Wazuh doesn't do any of that automatically in this setup (see the active-response note in Section 5.2), but `100100` is exactly the kind of high-confidence signal that would justify kicking those off, either manually or through an active-response script tied to this rule.
 
---
 
## 5. Strengths & Limitations
 
Section 1 puts Wazuh mainly in the **Identification** phase, since that's what my hands-on work actually demonstrates. But its feature set touches other phases too, even where I didn't build a full demo for them. Configuration Assessment (Section 3.2) supports **Preparation**, by catching misconfigurations before an incident even happens. Active response, which I scoped out below, supports **Containment**, since it'd act on an alert automatically instead of waiting on a human. File Integrity Monitoring, also scoped out, supports **Eradication** by catching follow-on changes an attacker makes after getting in, exactly the kind of thing a compromised account like `dmatute` could go do next. My own contribution here really sits right at the Identification/Containment boundary: `100100` identifies the compromise, and the response steps at the end of Section 4 are what Containment would look like next.
 
**Recovery** would build on that same response: going back and re-querying the Section 4.6 dashboard after the password reset and IP block to confirm no more `5763` or `100100` alerts are firing, basically verifying the endpoint's actually clean. **Lessons Learned** is honestly something this project already did, just not under that name. Sections 4.3 through 4.5 are literally me finding a real gap in the default ruleset and writing `100100` to close it, so the next person doesn't have to catch it by hand.
 
### 5.1 Strengths
 
**Real-time, correlated detection.** Wazuh checks events against the rule engine as they come in instead of just archiving them, and it can string multiple events together into one higher-confidence alert. I saw this directly: eight separate failed logins automatically got rolled into one brute-force alert (`5763`) using nothing but the default ruleset.
 
**Room to build past the defaults.** The out-of-the-box rules are a decent starting point, but they're a floor, not a ceiling. Building `100100` showed me the correlation tags (`if_sid`, `if_matched_sid`, `same_source_ip`) are flexible enough to actually patch a real gap: a successful login getting the same severity as any other one, even right after a brute-force attempt.
 
**MITRE context just comes with it.** Every rule that fired during this project, default or custom, already had MITRE technique/tactic labels attached. That's useful two ways: it speeds up triage for whoever's looking at it, and it gives you a shared vocabulary to explain the finding to people who don't know Wazuh's rule IDs but will recognize MITRE terminology.
 
**Compliance tagging is automatic.** The `5763` alert came pre-tagged with GDPR, HIPAA, PCI DSS, NIST 800-53, and TSC references, no extra work needed (Section 4.3). That kind of cross-referencing is usually its own tedious task; here it's just metadata attached the second the alert fires.
 
**`wazuh-logtest` catches mistakes before they matter.** I could test `100100` against real captured log lines before it ever touched live traffic (Section 4.5), which meant I could confirm the logic was right without risking a flood of false positives or, worse, it quietly not firing the next time the attack actually happened.
 
### 5.2 Limitations & Scope Tradeoffs
 
**Single-host, dual-role setup.** Because I only had 8GB of RAM on my host machine, VM1 had to be both the manager and the monitored "victim" endpoint (Section 2.10), instead of running the manager and a separately installed agent on two different machines the way a real deployment would. Reasonable call given the hardware, but it does mean this project doesn't show an actually independent agent deployment, agent-to-manager networking, or what monitoring multiple endpoints from one manager actually looks like.
 
**No active-response demo.** Wazuh can automatically block an attacking IP via firewall rule once a brute-force alert fires, that would've been a natural next step after `100100`. This maps to **Containment**, actually stopping the attacker instead of just recording what happened. I scoped it out on purpose to keep the project focused on detection and correlation rather than response, given the time I had.
 
**No File Integrity Monitoring demo.** Section 3.2 covers FIM as a core feature, especially relevant for catching post-compromise activity like backdoors or tampered binaries. That maps to **Eradication**: confirming an attacker's follow-on changes were actually found and removed, not just that the initial break-in got flagged. My scenario stopped at the successful login and didn't simulate any follow-on activity on the compromised account that FIM would've caught.
 
**Default rule severity gap (the main finding here).** The biggest limitation I found wasn't with my lab setup, it was with Wazuh's default ruleset itself: a successful SSH login gets treated identically (`5715`, level 3) whether it's a normal morning login or the direct result of a brute-force attack that just triggered a level-10 alert seconds earlier (Section 4.3). Nothing in the defaults connects those two. That's exactly the kind of blind spot a SOC analyst would have to catch manually, or, like I did, close with a custom rule.
 
**Environment-specific install headaches.** A few issues I ran into were specific to my setup (Apple Silicon, ARM64, UTM) rather than actual Wazuh problems, but worth noting since they ate real time:
- Wazuh's default install script rejects ARM64 outright, needed the version-specific 4.14 branch installer (Section 2.9).
- Ubuntu's guided LVM partitioning under-allocated disk space twice in a row, once inside the existing virtual disk and again after I'd already resized the virtual disk itself, each time failing the dashboard install or login with a confusing error instead of just saying "disk full."
- A GRUB reboot loop happened after my first Ubuntu install because I forgot to eject the installation ISO before rebooting.
None of these are actually Wazuh's fault, just the tradeoff of running it on Apple Silicon, which it wasn't originally built for. Free and open-source doesn't mean frictionless, and it's worth checking platform compatibility before you sink time into an install.
 
---
 
## 6. References
 
**Wazuh documentation**
- Wazuh Installation Guide: https://documentation.wazuh.com/current/installation-guide/index.html
- Wazuh Ruleset & Decoders Documentation: https://documentation.wazuh.com/current/user-manual/ruleset/index.html
- Wazuh `wazuh-logtest` Reference: https://documentation.wazuh.com/current/user-manual/reference/tools/wazuh-logtest.html
- Wazuh 4.14 ARM64-compatible installer: https://packages.wazuh.com/4.14/wazuh-install.sh
**MITRE ATT&CK Framework**
- MITRE ATT&CK, T1110 (Brute Force): https://attack.mitre.org/techniques/T1110/
- MITRE ATT&CK, T1078 (Valid Accounts): https://attack.mitre.org/techniques/T1078/
- MITRE ATT&CK, T1021 (Remote Services): https://attack.mitre.org/techniques/T1021/
**Virtualization & OS setup**
- UTM for macOS: https://mac.getutm.app/
- Ubuntu Server 24.04.4 LTS download (ARM64): https://ubuntu.com/download/server
- Kali Linux, official UTM installation guide: https://www.kali.org/docs/virtualization/install-utm-guest-vm/
- Kali Linux, installation and setup guide: https://www.kali.org/docs/installation/hard-disk-install/
**Attack tooling**
- THC-Hydra (SSH brute-force tool used in Section 4.2): https://github.com/vanhauser-thc/thc-hydra
**AI assistance**
- Claude (Anthropic): used throughout the project for troubleshooting help: https://claude.ai
---
 
✒️ **Author**
David Matute-Jimenez
