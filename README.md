## 4. Development Environment

### VM Setup
**Author: David**

## To Run This Project (Step-By-Step Guide)

### 4.1 Download and Install UTM

**Step 1:** Visit https://mac.getutm.app/

**Step 2:** Click the download button for macOS

**Step 3:** Open the downloaded `.dmg` file

**Step 4:** Drag UTM to your Applications folder

**Step 5:** Open Applications folder and launch UTM

**Step 6:** Grant any necessary permissions when prompted

*Screenshot: `17-utm-welcome-screen.png`*

---

### 4.2 Download Ubuntu Server 24.04.4 (ARM64)

**Step 7:** Visit https://ubuntu.com/download/server and select the **64-bit ARM (AArch64) server install image** — UTM on Apple Silicon requires ARM64, not the standard x86 ISO.

**Step 8:** Download Ubuntu Server **24.04.4 LTS** (Noble Numbat).

*Screenshot: `01-ubuntu-download-page.png`*

---

### 4.3 Deploy Ubuntu Server as VM1 (Wazuh Manager)

**Step 9:** In UTM, click **Create a New Virtual Machine** and select the Ubuntu Server ARM64 ISO as the boot image.

**Step 10:** Allocate hardware resources. Due to the 8GB RAM constraint on the host machine, this VM was configured with 5GB RAM, 3 CPU cores, and 30GB storage.

*Screenshot: `16-hardware-config.png`*

**Step 11:** Name the VM **VM1-Wazuh-Manager** and complete creation.

---

### 4.4 Install Ubuntu Server on VM1

**Step 12:** Boot the VM and select **Ubuntu Server** (not the minimized variant) as the installation type.

**Step 13:** Configure networking — DHCP auto-assigned IP `192.168.64.4`.

**Step 14:** Configure guided storage using the entire disk.

**Step 15:** Set up the user profile (username: `dmatute`).

**Step 16:** In SSH configuration, check **Install OpenSSH server** and leave **Allow password authentication over SSH** enabled — this is required later for the Hydra brute-force demo.

*Screenshot: `20-ssh-configuration-openssh.png`*

**Step 17:** Complete installation, remove the installation medium, and reboot.

> **Troubleshooting note:** The VM hit a GRUB reboot loop on first restart because the ISO wasn't ejected. Fixed by force-stopping the VM in UTM and manually clearing the CD/DVD drive in VM Settings before rebooting.

*Screenshot: `28-grub-reboot-loop-issue.png`*

**Step 18:** VM1 boots successfully into the installed system.

*Screenshot: `34-successful-boot-log.png`*

---

### 4.5 Deploy Kali Linux on UTM

**Step 19:** Follow Kali Linux's official UTM installation guide: https://www.kali.org/docs/virtualization/install-utm-guest-vm/

This guide will walk you through:
* Downloading the Kali Linux ARM64 image for UTM
* Creating a new virtual machine in UTM
* Allocating CPU, RAM, and storage resources
* Importing the Kali Linux image

### 4.6 Setup and Configure Kali Linux

**Step 20:** Follow Kali Linux's installation and setup guide: https://www.kali.org/docs/installation/hard-disk-install/

This guide will walk you through:
* Initial Kali Linux boot and configuration
* Partitioning and disk setup
* User account creation
* System updates and package configuration
* Desktop environment setup

### 4.7 Login to Kali Linux

**Step 21:** At the Kali Linux login screen, enter:
* **Username:** `kali`
* **Password:** `kali`

**Step 22:** Press Enter to login

---

### 4.8 Confirm Network Connectivity Between VMs

**Step 23:** With both VMs running in UTM's Shared Network mode, ping VM1 from Kali to confirm connectivity.

*Screenshot: `40-ping-kali-to-vm1-success.png`*

**Step 24:** Ping Kali from VM1 to confirm bidirectional connectivity.

*Screenshot: `42-ping-vm1-to-kali-success.png`*

**Step 25:** Confirm SSH access from the host Mac terminal into VM1.

*Screenshot: `39-ssh-login-success-vm1.png`*
