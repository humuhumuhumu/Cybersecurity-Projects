# Tracing an Intrusion: An End-to-End Threat Hunt with a SIEM

> *This writeup details my methodology for completing the "Hunting For Stuxbot" module from Hack The Box Academy.*

## 1. The Challenge: An Overview

In this investigation, I acted as a security analyst for a simulated enterprise network. My objective was to use a SIEM (Kibana) to analyze endpoint (Sysmon) and network (Zeek) logs to determine if the network had been compromised by a known malware campaign. The goal was to trace the full attack chain, from initial access to lateral movement.

---

## 2. Tools Used

* **SIEM**: Kibana / ELK Stack
* **Endpoint Telemetry**: Sysmon
* **Network Telemetry**: Zeek
* **Query Language**: KQL (Kibana Query Language)

---

## 3. Investigation & Methodology

The hunt was structured to follow a logical progression of the cyber kill chain, starting with the initial point of compromise.

### 3.1. Initial Access: Finding the Phishing Lure

The investigation began with the hypothesis that the compromise originated from a malicious OneNote file. I started by searching for the primary Indicator of Compromise (IOC), the filename `invoice.one`, in the Sysmon logs. I used `Event ID 11` (FileCreate) as it provides ground truth for file creation on the host.

```kql
event.code:11 AND file.name:invoice.one*
```

This query returned a hit, confirming the file was created on workstation `WS001` in the user Bob's `Downloads` directory. This served as the initial anchor point for the entire investigation.

![Kibana result showing the Sysmon Event ID 11 for invoice.one](https://i.imgur.com/your_image_placeholder_1.png)
*Caption: The initial hit in Kibana, confirming the creation of the malicious OneNote file on the endpoint.*

### 3.2. Execution: Unraveling the Process Chain

With the file on the system, the next logical question was, "Was it executed?" I hypothesized that the user would have opened the file with `ONENOTE.EXE`. Therefore, my next step was to hunt for any suspicious child processes created by the OneNote process.

```kql
event.code:1 AND process.parent.name:"ONENOTE.EXE"
```

This query revealed a clear and malicious process lineage. The logs showed that `ONENOTE.EXE` launched `cmd.exe`, which then executed a file named `invoice.bat` from a temporary directory. The final step in the chain was `invoice.bat` launching `powershell.exe` with a long, suspicious command line. This confirmed that the initial file was malicious and had successfully executed its first-stage payload.

![The process creation chain in Kibana](https://i.imgur.com/your_image_placeholder_2.png)
*Caption: The process creation chain in Kibana, showing how OneNote was used to launch PowerShell.*

### 3.3. C2 Identification: Pivoting from Endpoint to Network

The PowerShell command line clearly indicated it was attempting to download content from `pastebin.com`. To understand the full scope of the network activity, I needed to identify the attacker's Command and Control (C2) server.

I pivoted my investigation from endpoint logs to network telemetry, specifically the **Zeek** logs which capture DNS traffic. I built a query to find all DNS requests made by the compromised host, `WS001`, around the time of the PowerShell execution, while filtering out common noise.

```kql
source.ip:192.168.28.130 AND dns.question.name:* AND NOT dns.question.name:(*.google.com OR *.msftncsi.com)
```

This query immediately revealed DNS requests to a dynamic DNS provider, **`ngrok.io`**. Ngrok is a legitimate tunneling service, but it is frequently abused by attackers to mask their C2 traffic behind a trusted domain and bypass simple firewall rules, making this a high-confidence indicator of malicious activity.

### 3.4. Lateral Movement: Following the Attacker

Having confirmed C2, the final step was to determine if the attacker had moved to other systems. I searched for the hash of the downloaded payload (`default.exe`) across all hosts in the environment.

```kql
process.hash.sha256:018d37cbd3878258c29db3bc3f2988b6ae688843801b9abc28e6151141ab66d4
```
The query returned a hit on a second server, `PKI`. To understand how the payload was executed there, I examined the parent process of this event. The parent process was **`PSEXESVC.exe`**. This is the service executable for the legitimate Microsoft administration tool PsExec. Its presence here confirms that the attacker used PsExec as a "living-off-the-land" binary to pivot from `WS001` and execute code on the `PKI` server.

![Lateral Movement via PsExec](https://i.imgur.com/your_image_placeholder_6.png)
*Caption: The Sysmon log on the PKI server showing the payload being executed by PSEXESVC.exe, confirming lateral movement.*

---

## 4. Conclusion & Key Findings

This investigation successfully traced a multi-stage intrusion by using KQL queries to pivot between endpoint (Sysmon) and network (Zeek) data sources.

* **Initial Access**: The attacker gained entry via a malicious OneNote file (`invoice.one`).
* **Execution**: The attacker used a malicious batch script and PowerShell to download a second-stage payload.
* **Lateral Movement**: The attacker moved from `WS001` to the `PKI` server using the legitimate administration tool, PsExec.

This project demonstrates a complete investigative workflow, from forming a hypothesis and writing initial queries to pivoting between data sources and correlating events to understand the full scope of an attack. The investigation successfully answered all of the lab's core questions by identifying the attacker's TTPs and key IOCs, including the **C2 domain (`ngrok.io`)** and the **lateral movement tool (`PsExec`)**.
