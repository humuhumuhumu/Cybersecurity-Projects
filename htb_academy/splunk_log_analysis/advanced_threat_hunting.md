# Advanced Threat Hunting in Splunk: From TTPs to Statistics

> *This writeup details my methodology for completing the "Intrusion Detection With Splunk" module from Hack The Box Academy. The dataset was provided by HTB; the analysis and methodology are my own.*

## 1. The Challenge: An Overview

In this project, my objective was to practice two different advanced threat analysis methodologies in a large-scale Splunk environment (>500,000 events): first, hunting for known Tactics, Techniques, and Procedures (TTPs), and second, using statistical analysis to find unknown anomalies.

---

## 2. Tools Used

* **SIEM**: Splunk Enterprise
* **Query Language**: SPL (Search Processing Language)
* **Data Sources**: Sysmon, Windows Event Logs, Linux Syslog

---

## 3. Investigation & Methodology

### 3.1. TTP-Based Hunting: Finding an In-Memory Attack

My first hunt focused on finding evidence of an in-memory, fileless attack. I investigated Sysmon Process Access events (`Event ID 10`) targeting the LSASS process and specifically analyzed the `CallTrace` field for signs of shellcode injection.

```splunk
index="main" EventCode=10 TargetImage="*lsass.exe" CallTrace="*UNKNOWN*" | stats count by SourceImage
```
This query searches for processes accessing LSASS where the code making the call originates from an **`UNKNOWN`** memory region (not backed by a file on disk). This is a strong indicator of shellcode, and the query successfully identified `rundll32.exe` as the source of the attack.

![Splunk query results showing an UNKNOWN memory region in the CallTrace](https://i.imgur.com/your_image_placeholder_5.png)
*Caption: The Splunk query results showing a suspicious process accessing LSASS from an UNKNOWN memory region.*

### 3.2. Statistical Anomaly Detection

Next, I used statistical analysis to find novel threats. My goal was to find processes creating an anomalous number of remote threads (`Event ID 8`), a potential indicator of process injection.

```splunk
index="main" sourcetype="WinEventLog:Sysmon" EventCode=8
| stats count by SourceImage
| eventstats avg(count) as avg stdev(count) as stdev
| eval isOutlier=if(count > avg+(2*stdev), 1, 0)
| where isOutlier=1
```
This query uses `eventstats` to calculate the **average and standard deviation** for all processes, creating a dynamic baseline. It then uses `eval` to identify any process whose activity was **more than two standard deviations above the average**.

---

## 4. Lessons Learned

While building these advanced queries, some of my initial attempts failed because of a simple mistake: **case sensitivity**. My query for `CallTrace` initially didn't work until I corrected `unknown` to `UNKNOWN`. This was a critical lesson in the importance of precise syntax and paying close attention to the specific data schema when threat hunting.

---

## 5. Conclusion & Key Findings

This assessment demonstrates proficiency in two essential, complementary threat hunting methodologies.

* **TTP-Based Hunting**: The investigation into an LSASS memory dump successfully answered the lab's question by identifying **`rundll32.exe`** as the source of the in-memory attack.
* **Statistical Analysis**: The search for unusual behavior successfully answered the lab's question by identifying the **outlier process** creating an anomalously high number of remote threads.
