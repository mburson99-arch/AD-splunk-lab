# 🔧 Troubleshooting Journal
*This document details the technical challenges encountered during the lab build and the steps taken to resolve them. It demonstrates the diagnostic process for Active Directory, GPO, and SIEM configuration.*

## 🛑 Issue 1: Admin Locked Out by Hardening Policy
**Problem:**
After deploying a Group Policy Object (GPO) to "Block Command Prompt" to harden the workstations against standard users, the Administrator account on the Domain Controller was also locked out of CMD, preventing necessary maintenance.

**Diagnosis:**
* Attempted to run `gpupdate /force` but received an "Access Denied" error message indicating the policy had applied to the entire domain, including the Administrator.
* Realized the GPO scope was too broad.

**Solution:**
1. Accessed **Group Policy Management** via GUI (as CMD was blocked).
2. Located the `SEC_Workstation_Hardening` policy.
3. Modified the **Delegation** tab > **Advanced**.
4. Added the `Domain Admins` group and set the permission to **"Deny"** for "Apply Group Policy."
5. Restarted the Server to force the policy update without CMD access.
**Status:** Resolved.

---

## 🚫 Issue 2: Splunk Indexer Not Receiving Data
**Problem:**
After installing Universal Forwarders on client endpoints, no hosts appeared in the Splunk Search Head (`index=_internal`).

**Diagnosis:**
* Ran `Test-NetConnection 172.16.0.1 -Port 9997` on the client. Result: `True` (Network OK).
* Ran `netstat -an | findstr 9997` on the Server.
* **Observation:** The command returned no output, indicating the server was not listening on the port, despite the firewall being open.

**Solution:**
1. Logged into Splunk Web Interface.
2. Navigated to **Settings > Forwarding and Receiving > Configure Receiving**.
3. Manually added Port `9997` as a listening port.
4. Re-ran `netstat`, confirmed status `LISTENING`.
**Status:** Resolved.

---

## 👻 Issue 3: Duplicate GUIDs (Hosts overwriting each other)
**Problem:**
Only one workstation (`Win10-B`) was visible in Splunk at a time. If `Win10-A` connected, `Win10-B` would disappear.

**Diagnosis:**
* Suspected a GUID conflict caused by cloning the Virtual Machine *after* the Splunk agent was installed (or imperfect sysprep).
* Splunk identified both machines as the same entity due to identical `instance.cfg` identifiers.

**Solution:**
1. Stopped the Splunk Forwarder service on the affected client.
2. Deleted the file `C:\Program Files\SplunkUniversalForwarder\etc\instance.cfg`.
3. Restarted the service (`Restart-Service SplunkForwarder`), forcing the agent to generate a new, unique GUID.
**Status:** Resolved. Both hosts visible simultaneously.

---

## 📝 Issue 4: Missing Security Logs (Event 4625)
**Problem:**
Although hosts were connected, searching for `EventCode=4625` (Failed Login) returned "No Results Found," even after simulating brute-force attacks.

**Diagnosis:**
* Inspected the configuration directory: `C:\Program Files\SplunkUniversalForwarder\etc\system\local`.
* **Observation:** The `inputs.conf` file was missing. The agent knew *where* to send data (`outputs.conf`) but not *what* data to send.

**Solution:**
1. Created a manual `inputs.conf` file.
2. Defined stanzas for `[WinEventLog://Security]`, `[WinEventLog://System]`, and `[WinEventLog://Application]`.
3. Deployed the file via the internal SMB share (`Lab_Transfer`) to bypass clipboard restrictions on the Virtual Machine.
4. Restarted the Forwarder service.
**Status:** Resolved. Failed logins immediately populated the dashboard.

---

## 🖥️ Issue 5: Remote Desktop (RDP) Connection Refused
**Problem:**
Unable to establish a Remote Desktop session from the Server (`DC01`) to the Client (`Win10-A`). The connection timed out with a generic error, and standard `Ping` requests (ICMP) also failed.

**Diagnosis:**
* **Firewall Check:** Ran `netsh advfirewall set allprofiles state off` on the client. `Ping` requests began working, but RDP still failed.
* **Service Check:** Ran `netstat -an | findstr 3389` on the client. Confirmed the service was `LISTENING`.
* **Network Path:** Ran `Test-NetConnection 172.16.0.10 -Port 3389` from the Server. Result was `True`, confirming the network path was open.
* **Root Cause:** The connection was being blocked by **Network Level Authentication (NLA)**, which was rejecting the initial handshake in the virtualized lab environment.

**Solution:**
1.  **Forced RDP On:** Modified the registry key `fDenyTSConnections` to `0` to ensure the Terminal Server service was active.
2.  **Bypassed NLA:** Ran the following command to disable the NLA requirement:
    ```cmd
    reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v UserAuthentication /t REG_DWORD /d 0 /f
    ```
3.  **Verification:** Successfully connected via `mstsc` from the Domain Controller.
**Status:** Resolved.

---

## 📂 Issue 6: GPO Drive Map Not Applying
**Problem:**
The "Common Drive Map" Group Policy intended to map the `Z:` drive for all users failed to appear on client workstations, despite the GPO being linked correctly.

**Diagnosis:**
* **Scope Check:** The GPO contained "User Configurations" but was linked to a "Computer" Organizational Unit.
* **Syntax Check:** Verified the UNC path by manually typing it into the `Run` box.
* **Discovery:** The shared folder name (`Software Repo`) contained a space, but the GPO path was originally written as (`SoftwareRepo`).

**Solution:**
1.  **Loopback Processing:** Enabled "Loopback Processing" (Merge Mode) in Computer Policies to force User settings to apply when logging into these specific workstations.
2.  **Path Correction:** Updated the GPO Drive Map properties to reflect the correct UNC path: `\\DC01\Software Repo`.
3.  **Validation:** Ran `gpupdate /force` and verified the `Z:` drive appeared upon next login.
**Status:** Resolved.
