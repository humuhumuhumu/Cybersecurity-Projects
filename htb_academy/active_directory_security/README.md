# Deconstructing the Domain: An Analysis of Active Directory Attack Paths

> *This writeup summarizes a series of hands-on labs from Hack The Box Academy focused on Active Directory security. The scenarios and environments were provided by HTB; the analysis and execution are my own.*

In this series of projects, my objective was to analyze a suite of common Active Directory (AD) attack techniques. For each technique, I performed the attack in a controlled lab and then pivoted to a defensive role to identify the specific forensic artifacts left behind.

---

## Tools Used

* **Reconnaissance**: BloodHound, SharpHound, PowerView
* **Exploitation**: Mimikatz, Rubeus, Certify.exe, PowerSploit, Impacket
* **Analysis**: Windows Event Viewer

---

## Analysis Walkthrough

### 1. Reconnaissance: Mapping Attack Paths with BloodHound

The investigation began with reconnaissance. I executed the **`SharpHound.exe`** collector on a domain-joined host to gather data on users, groups, and ACLs. By importing this data into the **BloodHound** GUI, I was able to visualize complex permission relationships and automatically identify privilege escalation paths.

```powershell
.\SharpHound.exe -c All
```

![BloodHound UI showing an attack path to Domain Admins](./images/blooudhound_attack_path.png)
*The BloodHound UI showing a clear attack path to Domain Admins.*

### 2. Credential Access: Kerberos & AD CS Exploitation

I investigated multiple TTPs for stealing credentials:
* **Kerberoasting**: Used `Rubeus` to exploit a service account with a weak password, requesting a TGS ticket and exporting its hash for offline cracking.

    ```powershell
    .\Rubeus.exe kerberoast /outfile:hashes.txt
    ```

* **AD CS Abuse (ESC1)**: Exploited a misconfigured certificate template that allowed any user to specify an alternate name. I used `Certify.exe` to request a certificate as the `Administrator` user.

    ```powershell
    .\Certify.exe request /ca:PKI.eagle.local\eagle-PKI-CA /template:UserCert /altname:Administrator
    ```

For each attack, I then connected to the Domain Controller and analyzed the Security Log to find the key defensive artifacts, such as **Event ID 4769** (Kerberoasting) and **Event ID 4886** (Malicious Certificate Issuance).

### 3. Full Domain Compromise: The Golden Ticket

The ultimate goal for many AD attackers is the `krbtgt` account hash. I demonstrated a full domain compromise by first using `mimikatz` to perform a **DCSync** attack.

```
mimikatz # lsadump::dcsync /domain:eagle.local /user:krbtgt
```

With the `krbtgt` hash, I then used `mimikatz` again to forge a **Golden Ticket**, providing persistent, high-level access to the entire domain.

---

## 4. Lessons Learned

When first investigating the AD CS abuse, I focused on finding logs related to the user who requested the certificate. However, the true indicator of compromise was not the request itself, but the attributes *within* the issued certificate, specifically the Subject Alternative Name (SAN) that allowed for impersonation. This taught me that for complex attacks, I need to analyze the full details of the event, not just the high-level metadata.

---

## 5. Conclusion & Key Findings

This series of exercises provided deep, practical insight into modern AD attacks.

* **Key Finding 1**: Misconfigured ACLs, certificate templates, and delegation settings are critical and often overlooked paths to full domain compromise.
* **Key Finding 2**: The investigation successfully answered the lab's questions, culminating in the extraction of the domain's **`krbtgt` NTLM hash**, the key to creating a Golden Ticket.