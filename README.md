# Wazuh IR Lab: Detecting SSH Brute-Force Attacks on a Compromised Endpoint

**Tool:** Wazuh (open-source SIEM/XDR)
**Author:** David Matute-Jimenez
**Course:** BCYB644-Intro-to-Inform-and-Cybersecurity

---

## 1. Tool Overview

Wazuh is a free, open-source security platform that combines SIEM (Security Information and Event Management) and XDR (Extended Detection and Response) features. Basically, it collects logs from whatever systems you're monitoring, runs those logs against a rule engine, and generates alerts in real time when something looks off.

It's made up of three main pieces: an **agent** that sits on each monitored machine and forwards log data, a central **manager** that decodes that data and checks it against rules, and an **indexer/dashboard** combo that stores the alerts and gives you a web interface to actually look at them.

What Wazuh is really doing is taking messy raw logs and turning them into something a person can act on — catching repeated failed logins before they become an actual breach, giving a team one place to check instead of ten different systems, tagging alerts with MITRE ATT&CK info so they mean something, and letting you write your own rules for anything the defaults don't catch.

That core function — raw logs to actionable alert — puts Wazuh mainly in the **Identification** phase of the incident response lifecycle, which is exactly what this project shows off in Section 4: Wazuh catching an SSH brute-force attack against a simulated victim machine and generating an alert for it. That's not the only phase it touches though. Section 5 gets into how some of the other features covered in Section 3 — Configuration Assessment, active response, File Integrity Monitoring — extend Wazuh into Preparation, Containment, Eradication, and even Recovery and Lessons Learned, even though this project's hands-on demo stuck to Identification.

---

## 2. Tool Requirements, Setup & Workflow

### VM Setup
**Author: David**

## To Run This Project (Step-By-Step Guide)

### 2.1 Download and Install UTM

**Step 1:** Go to https://mac.getutm.app/, download UTM for macOS, open the `.dmg`, and drag it into Applications. Launch it and click through any permission prompts.

---

### 2.2 Download Ubuntu Server 24.04.4 (ARM64)

**Step 2:** Head to https://ubuntu.com/download/server and grab the **64-bit ARM (AArch64) server install image** — UTM on Apple Silicon needs ARM64, the regular x86 ISO won't work. I used Ubuntu Server **24.04.4 LTS** (Noble Numbat).

---

### 2.3 Deploy Ubuntu Server as VM1 (Wazuh Manager)

**Step 3:** In UTM, hit **Create a New Virtual Machine** and point it at the Ubuntu Server ARM64 ISO.

**Step 4:** Set your hardware. My host machine only has 8GB of RAM, so I gave this VM 5GB RAM, 3 CPU cores, and 30GB storage.

![VM1 Hardware Configuration](ProjectImages/utm-vm-hardware-config.png)

**Step 5:** Named it **VM1-Wazuh-Manager** and finished setup.

---

### 2.4 Install Ubuntu Server on VM1

**Step 6:** Boot the VM, pick **Ubuntu Server** (not the minimized version), and go through setup — networking grabbed `192.168.64.4` via DHCP, guided storage used the whole disk, and I set the username to `dmatute`.

**Step 7:** When it hits SSH configuration, check **Install OpenSSH server** and leave **Allow password authentication over SSH** on — I needed this later for the Hydra demo.

![SSH Configuration](ProjectImages/ssh-configuration-openssh.png)

**Step 8:** Finish install, pull the installation medium, reboot.

> **Ran into an issue here:** hit a GRUB reboot loop on the first restart because I forgot to eject the ISO. Fixed it by force-stopping the VM in UTM and manually clearing the CD/DVD drive in settings before rebooting.

![GRUB Reboot Loop Issue](ProjectImages/grub-reboot-loop-issue.png)

**Step 9:** VM1 boots fine after that.

---

### 2.5 Deploy Kali Linux on UTM

I set up Kali following their official docs — Kali's install process is well-maintained and doesn't have the ARM64 headaches Ubuntu Server gave me, so I'm not going to re-document the whole thing here. Their guide covers it better than I would.

**Step 10:** Follow Kali's official UTM install guide: https://www.kali.org/docs/virtualization/install-utm-guest-vm/

Covers downloading the ARM64 image, creating the VM, allocating resources, and importing it.

### 2.6 Setup and Configure Kali Linux

**Step 11:** Follow Kali's setup guide: https://www.kali.org/docs/installation/hard-disk-install/

Covers first boot, partitioning, creating a user, updates, and the desktop environment.

### 2.7 Login to Kali Linux

**Step 12:** Login screen — username `kali`, password `kali`.

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

It rejected the system right away, even though VM1 is genuinely 64-bit — just ARM64, not x86_64:
