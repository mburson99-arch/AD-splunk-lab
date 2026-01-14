# AD-splunk-lab
Active directory home lab with splunk siem intergration and help desk simulations
# 🛡️ Active Directory Home Lab & Splunk SIEM Implementation

## 📝 Project Overview
This project simulates a small corporate network environment designed to mimic real-world IT operations and Security Operations Center (SOC) monitoring. The goal was to build a domain from scratch, harden the security posture, and implement a centralized logging solution (SIEM) without relying on external internet tools for internal deployment.

The lab consists of a **Domain Controller (Windows Server 2022)** and **two Client Workstations (Windows 10)**, all configured within a virtualized air-gapped network.

## 🎯 Key Objectives
* **Infrastructure:** Deploy a Windows Active Directory (AD) environment with DNS/DHCP.
* **Security Hardening:** Enforce "Least Privilege" and attack surface reduction via Group Policy (GPO).
* **Software Deployment:** Simulate an enterprise "offline" deployment using internal SMB Network Shares.
* **SIEM Telemetry:** Ingest and analyze security logs (Event ID 4625) using Splunk Enterprise and Universal Forwarders.
* **IT Support:** Simulate Tier 1/2 Help Desk tickets including account lockouts, remote access, and storage management.

## 🛠️ Tools & Technologies
* **Hypervisor:** Oracle VirtualBox
* **OS:** Windows Server 2022, Windows 10 Enterprise
* **Identity:** Active Directory Domain Services (AD DS)
* **SIEM:** Splunk Enterprise (Indexer), Splunk Universal Forwarder (Agent)
* **Network:** SMB File Sharing, PowerShell, Firewall Management, RDP
* **Protocols:** DNS, DHCP, TCP/IP, ICMP

---

## 🏗️ Architecture Setup
* **DC01 (Server):** Acts as the Domain Controller, DNS Server, and Splunk Indexer. hosted a rigid "Software Repository" share to distribute installers to clients.
* **Win10-A (Client):** HR Workstation (User: Angela).
* **Win10-B (Client):** IT Workstation (User: John).
* **Network:** All devices connected via an internal Virtual LAN (172.16.x.x) with strict connectivity tests performed via PowerShell (`Test-NetConnection`).

---

## 🚀 Phase 1: Deployment & Hardening
1.  **Domain Configuration:** Promoted Server 2022 to a Domain Controller (`BursonLabs.local`) and configured OU structures (IT, HR).
2.  **PowerShell Automation:** Utilized PowerShell scripts to bulk-create users and assign them to specific organizational units.
3.  **GPO Hardening:**
    * Created a "Restricted Admin" policy.
    * **Blocked Command Prompt (CMD)** execution for standard users to prevent unauthorized scripts.
    * Removed standard users from the Local Administrators group to enforce Least Privilege.

## 📡 Phase 2: Internal Software Deployment (The "Offline" Method)
Instead of allowing workstations to download software from the internet (which introduces risk), I implemented a secure supply chain:
1.  Downloaded the Splunk Universal Forwarder installer on the host machine.
2.  Transferred it to a secured **SMB Network Share** on the Domain Controller (`\\DC01\SoftwareRepo`).
3.  Connected clients (Win10-A/B) to the share and installed the agent over the internal network.
4.  **Challenge:** UAC blocked the installation for standard users.
5.  **Solution:** Used Domain Admin credentials to elevate privileges and complete the install, verifying that "Least Privilege" policies were active.

## 🔍 Phase 3: Splunk SIEM Configuration & Troubleshooting
After installing the forwarders, the Server was not receiving logs. I performed the following troubleshooting steps:
1.  **Connectivity Check:** Used `Test-NetConnection -Port 9997` to confirm the Forwarders could reach the Indexer.
2.  **Firewall Fix:** Configured Inbound Rules on the Server to allow TCP traffic on port 9997.
3.  **Manual Configuration (The Critical Fix):**
    * Identified that the `inputs.conf` file was missing from the Forwarder installation.
    * Manually authored and deployed a custom `inputs.conf` to both endpoints to explicitly target **Security**, **System**, and **Application** logs.
    * *See attached code snippet `inputs.conf` for configuration details.*

## 🛡️ Phase 4: Attack Simulation & Detection
To verify the SIEM was operational, I simulated a brute-force attack:
1.  Attempted multiple failed logins on `Win10-A` (Angela) to generate noise.
2.  Queried Splunk for **Event Code 4625** (An account failed to log on).
3.  **Result:** Successfully visualized the attack time, target user, and source workstation on the Splunk Dashboard.

## 🤝 Phase 5: Help Desk & IT Operations
**Objective:** Simulate Tier 2 Support tasks including remote administration and storage management.

* **Remote Support (RDP):**
    * Enabled Remote Desktop Protocol on endpoints to facilitate remote troubleshooting.
    * Diagnosed and resolved connectivity issues where **Network Level Authentication (NLA)** prevented session establishment.
    * Utilized **Registry Keys** (`reg add`) to force configuration changes when GUI access was restricted.
* **Storage Management:**
    * Performed disk partitioning on Windows 10 endpoints using **Disk Management** (`diskmgmt.msc`).
    * Shrunk the primary system volume (C:) to reclaim unallocated space without data loss.
    * Initialized, formatted (NTFS), and mounted a new logical volume (`E:\`) for dedicated department data storage.

## 💻 Phase 6: Identity Management & Automation
**Objective:** Resolve common Tier 1 support tickets and automate user environment configuration.

* **Identity Management (Active Directory):**
    * Configured **Account Lockout Policies** (Threshold: 5 attempts) to mitigate brute-force risks.
    * Simulated a "User Lockout" scenario and utilized ADUC tools to unlock user accounts and reset credentials.
    * Enforced security best practices by checking "User must change password at next logon."
* **Automated Drive Mapping (GPO):**
    * Created a "Common Drive Map" policy to automatically mount the internal software share as the `Z:` drive for all employees.
    * Utilized **Loopback Processing** to ensure user policies applied correctly within the Computer OU structure.

---

## 📸 Screenshots & Evidence

### 1. Network Architecture
*VirtualBox Manager showing the Domain Controller and Client Workstations running simultaneously.*
<img width="955" height="513" alt="Github project Splunk picture of all three locatable" src="https://github.com/user-attachments/assets/9b78ad3c-1b48-4ec7-a33a-c40e5adb4cd4" />


### 2. Splunk SIEM Dashboard
*Visualizing the failed login attempts (Event 4625) from the "Analysis" phase.*
<img width="949" height="501" alt="Screenshot github BOTH eventcode screenlock wrong password" src="https://github.com/user-attachments/assets/0e1cc26d-b25c-4f8c-9f37-4805e2f0e4f1" />


### 3. Remote Desktop (RDP) Success
*The Domain Controller successfully connected to the Client Desktop after fixing the NLA/Firewall issue.*
<img width="955" height="506" alt="RDP to Win10-A via server side using powershell" src="https://github.com/user-attachments/assets/b3856bc6-7765-4a40-8788-5348088d3aec" />


### 4. Disk Management (Storage)
*The new "IT_DATA (E:)" drive created via partitioning.*
<img width="695" height="449" alt="Managed Storage subsystems by resizing partitions and initializing new NTFS Volumes for Dedicated storage" src="https://github.com/user-attachments/assets/0d823128-5c2e-4ad8-a145-220125350816" />


### 5. Group Policy Drive Map
*The "Company Data (Z:)" drive automatically appearing in File Explorer.*
<img width="943" height="509" alt="Implelemented GPO Prefrences to map a shared network drives to ensureing seamless data access for end users" src="https://github.com/user-attachments/assets/76711c71-01ee-4ba6-9e09-f048ac717633" />

