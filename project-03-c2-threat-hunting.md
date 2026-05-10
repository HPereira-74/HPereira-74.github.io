# Project 3: Advanced Command & Control (C2) Operations & Threat Hunting

## Objective
To architect an offensive engagement against a hardened, localized Windows 11 endpoint using an enterprise-grade Command and Control (C2) framework. The objective encompasses payload weaponization, defense evasion via Living-off-the-Land (LotL) techniques, simulated post-exploitation (persistence and ransomware behavior), and a comprehensive Blue Team detection analysis utilizing the Wazuh SIEM and Sysmon telemetry.

---

## Architecture & Tool Selection Rationale

Before executing the engagement, specific tools were selected to accurately mirror modern Advanced Persistent Threat (APT) behaviors:

* **Attacker Infrastructure (Kali Linux):** Selected as the industry-standard offensive operating system, deployed on a localized virtual network (`192.168.74.30`) to ensure complete isolation from production networks.
* **C2 Framework (Sliver by BishopFox):** While legacy frameworks like Metasploit are heavily signature-tracked by modern EDRs, Sliver is a modern, Golang-based C2 actively utilized by real-world threat actors (such as APT29). It supports cross-compilation, advanced memory injection, and encrypted C2 channels (mTLS, WireGuard), making it an ideal platform for simulating contemporary cyber threats.
* **Telemetry Pipeline (Sysmon + Wazuh):** Relying solely on native Windows Event Logs provides insufficient visibility into process creation and memory manipulation. Sysmon (Event IDs 1 and 11) was utilized to provide kernel-level process tracking, directly forwarding telemetry to the Wazuh Manager for analysis.

---

## Methodology & Execution Lifecycle

### Phase 1: C2 Initialization & Weaponization
The offensive infrastructure was staged on the Kali Linux node. An HTTP listener was established on Port 80. 
* **Strategic Rationale:** While HTTPS is standard for evasion, plaintext HTTP was intentionally selected for this lab environment to allow for deep packet inspection and cleartext telemetry analysis within the SIEM, enabling a better understanding of the C2 beaconing structure.

A custom Windows executable (`amd64`) implant was compiled via Sliver, hardcoded with the attacker IP address and callback port.


### Phase 2: Defense Evasion & Payload Delivery
To deliver the payload to the victim endpoint (`192.168.74.20`), a temporary Python web server was staged on the attacker machine. 

* **Living-off-the-Land (LotL) Delivery:** Instead of downloading the payload via a web browser (which would trigger Windows SmartScreen, mark the file with a Mark-of-the-Web (MotW), and leave significant forensic artifacts), I utilized the native Windows `PowerShell.exe` process. By executing `Invoke-WebRequest` via the command line, the payload was pulled directly to the disk. This mimics real-world adversary techniques designed to bypass user-interaction requirements and browser-based security controls.

### Phase 3: Post-Exploitation & Impact
Upon execution of the payload, a persistent session successfully checked into the Sliver C2 server. 

**Overcoming Environmental Instability:**
During post-exploitation, attempts to establish a continuous, interactive shell tunnel hung indefinitely. This was diagnosed as interference from background Windows OS updates and network state changes. 
* **The Pivot:** Advanced attackers avoid interactive shells as they are highly unstable and generate excessive network noise. To bypass the tunnel disruption, I pivoted to Sliver's `execute` module (`execute -o cmd.exe /c`). This passed the malicious instructions directly into Windows memory as one-shot executions, successfully stabilizing the attack path.


With stable execution established, I simulated two distinct MITRE ATT&CK techniques:

1.  **Persistence (T1136 - Account Creation):** Executed `net localgroup administrators HACKER /add`. 
    * *Rationale:* Attackers provision rogue administrative accounts to guarantee continued access to the environment in the event the primary C2 payload is discovered and remediated by the Blue Team.
2.  **Impact / Ransomware Simulation (T1490 - Inhibit System Recovery):** Executed `vssadmin delete shadows /all /quiet`. 
    * *Rationale:* This is a primary behavior of modern ransomware strains (e.g., LockBit, Conti). By deleting Volume Shadow Copies, the attacker ensures the victim cannot restore encrypted files from local backups.

---

## Detection Engineering & SIEM Analysis (Blue Team)

Following the Red Team execution, I pivoted to the Wazuh SIEM to analyze the Sysmon telemetry and validate the detection pipeline. The pipeline successfully captured the entire attack chain.

### 1. Initial Access & Tool Transfer (Event ID 11)
Despite bypassing the browser, the SIEM detected the initial PowerShell staging. Wazuh captured `powershell.exe` dropping a temporary script policy file during the execution of `Invoke-WebRequest`.
* **Severity:** Wazuh correctly classified this as a Level 15 (Critical) Alert.
* **MITRE Mapping:** T1105 (Ingress Tool Transfer).


### 2. Persistence Mechanism Detection (Event ID 1)
Sysmon Event ID 1 (Process Creation) recorded the exact command line execution used to create the backdoor administrator account (`net localgroup administrators HACKER /add`) running at a High Integrity level.
* **Analysis:** Wazuh flagged this as a Level 3 (Warning) Alert for "Account Discovery." While accurately logged, this highlights a common SIEM tuning requirement: default rulesets often grade administrative commands as low-severity because legitimate IT staff use them. This identifies an opportunity to write custom detection rules that elevate the severity if `net.exe` is spawned by an unknown or untrusted parent process.


### 3. Ransomware Behavior Detection (Event ID 1)
The SIEM successfully captured the `vssadmin` shadow copy deletion. The logs explicitly showed the parent process (`cmd.exe /c`) executing the `vssadmin delete shadows` instruction, directly validating the success of the one-shot memory execution pivot used during the attack phase.
* **Analysis:** Similar to the persistence detection, Wazuh flagged this as a Level 3 alert ("Suspicious Windows cmd shell execution"). This further emphasizes the necessity of dedicated Detection Engineers in a SOC environment to write specific rules for high-fidelity indicators of compromise (IoCs) like `vssadmin`.


---

## Conclusion
This engagement successfully validated the complete lifecycle of a cyber attack within a controlled environment. It demonstrated the practical application of C2 infrastructure, the necessity of adaptable tradecraft (LotL, memory execution) to bypass modern OS hardening, and the critical importance of kernel-level telemetry (Sysmon) for accurate threat hunting and SIEM alerting.