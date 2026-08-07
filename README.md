# Wazuh IR Lab: Detecting SSH Brute-Force Attacks on a Compromised Endpoint

- **Tool:** Wazuh (open-source SIEM/XDR)
- **Author:** David Matute-Jimenez
- **Course:** BCYB644-Intro-to-Inform-and-Cybersecurity

---

## 1. Tool Overview

Wazuh is a free, open-source security platform that combines SIEM (Security Information and Event Management) and XDR (Extended Detection and Response) capabilities. It collects log data from monitored endpoints, correlates that data against a rule engine, and generates real-time alerts when suspicious or malicious activity is detected.

Wazuh is built around three main components: a lightweight **agent** installed on each monitored endpoint that collects and forwards log data; a central **manager** that analyzes that data against its rule engine and decoders; and an **indexer and dashboard** that store alerts and logs and provide a web interface for investigation.

Wazuh solves several core problems in cybersecurity: it enables near real-time threat detection (such as flagging repeated failed login attempts as a possible brute-force attack), gives analysts centralized visibility across an environment instead of manually checking individual system logs, maps alerts to MITRE ATT&CK techniques for deeper context, and supports custom detection rules for threats the default ruleset doesn't cover.

In the incident response lifecycle, Wazuh is most directly tied to the **Identification** phase — it turns raw log data into an actionable alert an analyst can investigate. This project demonstrates that role directly: Wazuh detects an SSH brute-force attack against a simulated victim endpoint, generates an alert, and supports the follow-up analysis (custom rule creation, MITRE mapping) that comes after initial detection.

---

## 2. Tool Requirements, Setup & Workflow

### VM Setup
**Author: David**

## To Run This Project (Step-By-Step Guide)

### 2.1 Download and Install UTM

**Step 1:** Visit https://mac.getutm.app/

**Step 2:** Click the download button for macOS

**Step 3:** Open the downloaded `.dmg` file

**Step 4:** Drag UTM to your Applications folder

**Step 5:** Open Applications folder and launch UTM

**Step 6:** Grant any necessary permissions when prompted

![UTM Welcome Screen](WazuhImages/04-utm-welcome-screen.png)

---

### 2.2 Download Ubuntu Server 24.04.4 (ARM64)

**Step 7:** Visit https://ubuntu.com/download/server and select the **64-bit ARM (AArch64) server install image** — UTM on Apple Silicon requires ARM64, not the standard x86 ISO.

**Step 8:** Download Ubuntu Server **24.04.4 LTS** (Noble Numbat).

![Ubuntu Download Page](WazuhImages/01-ubuntu-download-page.png)

---

### 2.3 Deploy Ubuntu Server as VM1 (Wazuh Manager)

**Step 9:** In UTM, click **Create a New Virtual Machine** and select the Ubuntu Server ARM64 ISO as the boot image.

**Step 10:** Allocate hardware resources. Due to the 8GB RAM constraint on the host machine, this VM was configured with 5GB RAM, 3 CPU cores, and 30GB storage.

![VM1 Hardware Configuration](WazuhImages/05-utm-vm-hardware-config.png)

**Step 11:** Name the VM **VM1-Wazuh-Manager** and complete creation.

---

### 2.4 Install Ubuntu Server on VM1

**Step 12:** Boot the VM and select **Ubuntu Server** (not the minimized variant) as the installation type.

**Step 13:** Configure networking — DHCP auto-assigned IP `192.168.64.4`.

**Step 14:** Configure guided storage using the entire disk.

**Step 15:** Set up the user profile (username: `dmatute`).

**Step 16:** In SSH configuration, check **Install OpenSSH server** and leave **Allow password authentication over SSH** enabled — this is required later for the Hydra brute-force demo.

![SSH Configuration](WazuhImages/23-ssh-configuration-openssh.png)

**Step 17:** Complete installation, remove the installation medium, and reboot.

> **Troubleshooting note:** The VM hit a GRUB reboot loop on first restart because the ISO wasn't ejected. Fixed by force-stopping the VM in UTM and manually clearing the CD/DVD drive in VM Settings before rebooting.

![GRUB Reboot Loop Issue](WazuhImages/28-grub-reboot-loop-issue.png)

**Step 18:** VM1 boots successfully into the installed system.

![Successful Boot Log](WazuhImages/34-successful-boot-log.png)

---

### 2.5 Deploy Kali Linux on UTM

Kali Linux was deployed as the attacker VM following Kali's official documentation. Since Kali provides a well-maintained, standard installation process (unlike the ARM64-specific issues encountered with Ubuntu Server), this section is intentionally brief — the official guides cover the process more thoroughly than a re-documentation here would.

**Step 19:** Follow Kali Linux's official UTM installation guide: https://www.kali.org/docs/virtualization/install-utm-guest-vm/

This guide will walk you through:
* Downloading the Kali Linux ARM64 image for UTM
* Creating a new virtual machine in UTM
* Allocating CPU, RAM, and storage resources
* Importing the Kali Linux image

![Kali Linux VM Summary in UTM](WazuhImages/101-kali-vm-summary-utm.png)

### 2.6 Setup and Configure Kali Linux

**Step 20:** Follow Kali Linux's installation and setup guide: https://www.kali.org/docs/installation/hard-disk-install/

This guide will walk you through:
* Initial Kali Linux boot and configuration
* Partitioning and disk setup
* User account creation
* System updates and package configuration
* Desktop environment setup

### 2.7 Login to Kali Linux

**Step 21:** At the Kali Linux login screen, enter:
* **Username:** `kali`
* **Password:** `kali`

**Step 22:** Press Enter to login

![Kali Login Screen](WazuhImages/100-kali-login-screen.png)

**Step 23:** Confirm the VM is networked correctly by checking its assigned IP address.

![Kali IP Address Check](WazuhImages/41-kali-ip-address-check.png)

---

### 2.8 Confirm Network Connectivity Between VMs

**Step 24:** With both VMs running in UTM's Shared Network mode, ping VM1 from Kali to confirm connectivity.

![Ping Kali to VM1 Success](WazuhImages/40-ping-kali-to-vm1-success.png)

**Step 25:** Ping Kali from VM1 to confirm bidirectional connectivity.

![Ping VM1 to Kali Success](WazuhImages/42-ping-vm1-to-kali-success.png)

**Step 26:** Confirm SSH access from the host Mac terminal into VM1.

![SSH Login Success VM1](WazuhImages/39-ssh-login-success-vm1.png)

---

### 2.9 Wazuh Installation on VM1

*(TBD — ARM64 install script issue, disk sizing/LVM resize saga, service restart ordering fix, agent deployment)*

---

### 2.10 Architecture Note: VM1's Dual Role

**Note:** Due to hardware constraints, VM1 serves as both the Wazuh manager and the monitored endpoint (running its own Wazuh agent). In a production deployment, these roles would typically run on separate systems.

VM1 (`192.168.64.4`) therefore acts as both the infrastructure running Wazuh (manager, indexer, dashboard) *and* the simulated victim endpoint — the "compromised employee laptop" targeted by the SSH brute-force attack demonstrated in Section 4.

---

## 3. Core Features

*(TBD — Wazuh dashboard, agent-based monitoring, rule engine/decoders, wazuh-logtest, MITRE ATT&CK integration. Screenshot each as demoed.)*

---

## 4. Practical Application

*(TBD — incident narrative: disgruntled employee laptop compromised via weak SSH creds. Hydra brute-force attack steps + output. Wazuh alerts firing, rule IDs. Custom rule + wazuh-logtest validation. MITRE ATT&CK T1110 mapping.)*

---

## 5. Strengths & Limitations

*(TBD — Strengths: real-time detection, customizable rules, MITRE integration. Limitations/scope tradeoffs: no active-response demo, no FIM demo, RAM constraint rationale — including VM1's dual manager/endpoint role (see Section 2.10). Lessons learned: ARM64 install issue, disk sizing, service restart ordering, GRUB reboot loop.)*

---

## 6. References

*(TBD — Wazuh docs, MITRE ATT&CK, UTM/Ubuntu/Kali install guides used)*
