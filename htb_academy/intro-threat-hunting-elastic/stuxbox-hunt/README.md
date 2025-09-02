# Tracing an Intrusion: An End-to-End Threat Hunt in an ELK Stack

> This write-up covers the "Hunting for Stuxbot" and "Skills Assessment" sections from the "Introduction to Threat Hunting & Hunting With Elastic" modules on Hack The Box Academy.

## 1. Objective

In this project, my objective was to trace a full attack chain in a simulated enterprise network. Starting with a single indicator, I used a SIEM (Kibana) to pivot between endpoint and network logs, building a complete picture of an intrusion from initial access to lateral movement.

## 2. Tools & Environment

* **SIEM**: Kibana / ELK Stack
* **Endpoint Telemetry**: Sysmon
* **Network Telemetry**: Zeek
* **Query Language**: KQL (Kibana Query Language)

---

## 3. The Hunt: From a Single IOC to a Full Attack Chain

### 3.1. The Lead: Finding the Phishing Lure

The hunt began with a single Indicator of Compromise (IOC): the filename `invoice.one`. To confirm its presence on the network, I queried **Sysmon** logs for `Event ID 11 (FileCreate)`. This is the most direct way to verify if and where a file was written to disk.

The query immediately returned a hit on workstation `WS001`, confirming the file was successfully downloaded by a user.


*Caption: The initial Kibana query confirms the creation of `invoice.one` on `WS001`.*

### 3.2. The Pivot: Uncovering the C2 Channel

With the point of entry confirmed, the next logical step was to investigate the network traffic from `WS001` around the time of the file creation. I pivoted from the endpoint logs (Sysmon) to the network logs (**Zeek**) to analyze DNS traffic (`dns.query`).

This revealed suspicious queries from `WS001` to a dynamic DNS provider, `ngrok.io`. Attackers frequently abuse services like ngrok to quickly stand up and tear down Command and Control (C2) infrastructure, making this a high-confidence indicator of malicious activity.

### 3.3. The Kill Chain: Tracing Local Execution

Returning to the Sysmon logs on `WS001`, I traced the process execution chain originating from the malicious OneNote file. By correlating parent and child process IDs, I built a clear picture of the execution flow:

1.  `ONENOTE.EXE` (The user opens the lure)
2.  `cmd.exe` (OneNote spawns a command prompt)
3.  `powershell.exe` (The command prompt launches PowerShell to download the next stage)

This three-step process is a classic TTP (Technique, Tactic, and Procedure) for initial code execution via malicious documents.

### 3.4. The Spread: Confirming Lateral Movement

The final piece of the puzzle was to determine if the attacker had moved beyond the initial workstation. I took the hash of the second-stage payload downloaded by PowerShell and searched for it across all hosts in the SIEM.

This search returned a hit on a second server, `PKI`. Critically, the parent process that created this file on `PKI` was `PSEXESVC.exe`. This is the service executable for **PsExec**, a legitimate systems administration tool that is heavily abused by attackers for lateral movement. The finding confirmed the attacker had successfully compromised `WS001` and used it to move to the `PKI` server.

---

## 4. Conclusion & IOCs

This hunt successfully reconstructed a multi-stage intrusion by correlating disparate data sources. By pivoting between endpoint and network telemetry, I was able to connect individual events into a logical attack chain.

### Key Findings

* **Initial Access**: Attacker gained entry via a malicious OneNote file (`invoice.one`).
* **Execution**: The file spawned `cmd.exe` and `powershell.exe` to download a second-stage payload.
* **C2 Communication**: The payload communicated with a C2 server hosted on `ngrok.io`.
* **Lateral Movement**: The attacker moved from `WS001` to the `PKI` server using `PsExec`.

### Indicators of Compromise (IOCs)

* **Filename**: `invoice.one`
* **C2 Domain**: `*.ngrok.io`
* **Lateral Movement Tool**: `PsExec` (indicated by `PSEXESVC.exe`)
