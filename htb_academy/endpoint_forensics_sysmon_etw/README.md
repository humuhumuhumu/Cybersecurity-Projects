# Beyond the Logs: Detecting Evasive Threats with Sysmon & ETW

> *This writeup details my process for completing several endpoint analysis modules from Hack The Box Academy.*

In this project, my objective was to go beyond standard Windows logs to detect evasive threats. I used advanced endpoint telemetry from **Sysmon** and **Event Tracing for Windows (ETW)** to find evidence of attacks designed to bypass conventional logging.

---

## Tools Used

* **Endpoint Logging**: Sysmon, Event Tracing for Windows (ETW)
* **ETW Collection**: SilkETW
* **Analysis**: Windows Event Viewer
* **Attack Tools**: Mimikatz

---

## Analysis Walkthrough

### 1. Unmasking Parent PID Spoofing with ETW

My first task was to investigate a Parent PID Spoofing attack. I first confirmed that the standard **Sysmon `Event ID 1`** log was successfully deceived, showing the *wrong* parent process. To find the ground truth, I used **SilkETW** to collect raw data from the `Microsoft-Windows-Kernel-Process` ETW provider.

```powershell
.\SilkETW.exe -t user -pn Microsoft-Windows-Kernel-Process -ot file -p C:\windows\temp\etw.json
```

By comparing the two logs, I identified the discrepancy and uncovered the true parent process, successfully detecting the evasion technique.

![Side-by-side comparison of the incorrect Sysmon log and the correct ETW log](./images/compare_sysmon_etw)
*A side-by-side comparison of the incorrect Sysmon parent process with the correct ETW log, revealing the true parent.*

### 2. Finding Credential Dumping

Next, I hunted for credential dumping by monitoring for **Sysmon `Event ID 10` (ProcessAccess)** events. I used the built-in filter in Windows Event Viewer to search the Sysmon/Operational log for events where the `TargetImage` field contained **`lsass.exe`**. This allowed me to identify an unauthorized process (`mimikatz.exe`) accessing the LSASS memory, a critical indicator of a credential dumping attack.

---

## 4. Lessons Learned

When I first saw the Sysmon log for the Parent PID Spoofing attack, my initial analysis was that the trusted service itself (`spoolsv.exe`) might have been compromised. This sent me down a rabbit hole of investigating the service's file integrity. It was only after this dead end that I realized the log itself could be deceptive, which led me to investigate deeper telemetry with ETW. This taught me to always **question your data sources**.

---

## 5. Conclusion & Key Findings

This analysis demonstrates the critical need for deep endpoint visibility to detect modern threats.

* **Key Finding 1**: The investigation into an in-memory attack successfully answered the lab's question by identifying **`mimikatz.exe`** as the process performing the LSASS dump.
* **Key Finding 2**: The analysis of evasive techniques proved that while Sysmon can be deceived, raw **ETW** data provides a lower-level "ground truth" that is much more difficult for attackers to manipulate.