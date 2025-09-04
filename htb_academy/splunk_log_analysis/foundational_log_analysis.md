# Investigating with Splunk: A Log Analysis Primer

> *This writeup details my methodology for completing the "Splunk Fundamentals" section of the "Investigating with Splunk" module from Hack The Box Academy. The scenario and data were provided by HTB; the analysis and methodology are my own.*

## 1. The Challenge: An Overview

In this project, I acted as a junior security analyst tasked with querying a Splunk instance to answer specific investigative questions about user and system activity. The objective was to use the Search Processing Language (SPL) to efficiently parse and aggregate Windows event log data to find key insights.

---

## 2. Tools Used

* **SIEM**: Splunk Enterprise
* **Query Language**: SPL (Search Processing Language)
* **Data Sources**: Windows Event Logs

---

## 3. Investigation & Methodology

The investigation involved three separate analytical tasks to answer questions about the dataset.

### 3.1. Finding Outliers with `stats`

My first task was to find the account with the highest number of Kerberos authentication ticket requests. I started with a broad search and then used the `stats` command to aggregate the data.

```splunk
index="main" "Kerberos authentication ticket"
| stats count by Account_Name
| sort -count
```
This query counts all relevant events and groups them by the `Account_Name` field, immediately revealing the most active account, `waldo`, as the primary outlier.

![Splunk stats query for Kerberos requests](https://i.imgur.com/your_image_placeholder_1.png)
*Caption: The SPL query using the `stats` command to find the user with the most Kerberos requests.*

### 3.2. Finding Unique Values with `dedup`

The second task was to find the number of distinct computers the `SYSTEM` account had accessed. After filtering for successful logon events (`EventCode=4624`) by the `SYSTEM` account, I used the `dedup` command to isolate the unique computer names before counting them.

```splunk
index="main" EventCode=4624 Account_Name="SYSTEM"
| dedup ComputerName
| stats count
```
This method is far more efficient than manually counting and confirmed that the SYSTEM account had accessed **10** distinct computers.

### 3.3. Performing Time-Based Analysis

The final task was to identify which account had the most login attempts within a 10-minute window. This required a more advanced, time-bound query.

```splunk
index="main" EventCode=4624
| stats count, range(_time) as duration by Account_Name
| where duration <= 600
| sort -count
```
This query first calculates the `count` of logons and the time range (`duration`) for each account. The `where` clause then filters this down to only include users whose activity occurred within a 600-second (10-minute) span, revealing the correct user.

---

## 4. Conclusion & Key Findings

This series of exercises demonstrated proficiency in using core SPL commands to investigate security logs.

* **Key Finding 1**: Successfully used `stats` to aggregate data and identify outliers.
* **Key Finding 2**: Successfully used `dedup` to find and count unique values in a dataset.
* **Key Finding 3**: Successfully performed time-based analysis using `range()` and `where` to identify bursts of activity.
