 Mediroza General Hospital — Penetration Testing Report
==========================================================

### Black-box web application security assessment demonstrating a full attack chain from recon to a critical data breach

![Skill](https://img.shields.io/badge/Skill-Cybersecurity-red) ![Skill](https://img.shields.io/badge/Skill-Web%20Pentesting-red) ![Target](https://img.shields.io/badge/Target-medirozahospital.com-blue) ![OS](https://img.shields.io/badge/OS-Kali%20Linux%20v2026.2-informational?logo=kalilinux&logoColor=white) ![Risk](https://img.shields.io/badge/Overall%20Risk-CRITICAL-critical)

![Skill](https://img.shields.io/badge/Skill-SQL%20Injection-red) ![Skill](https://img.shields.io/badge/Skill-Password%20Cracking-red) ![Skill](https://img.shields.io/badge/Skill-OSINT%20%26%20Recon-red) ![Tool](https://img.shields.io/badge/Tool-exiftool-black) ![Tool](https://img.shields.io/badge/Tool-qpdf-black) ![Tool](https://img.shields.io/badge/Tool-curl%20%2F%20wget-black)

![Org](https://img.shields.io/badge/Organization-Networkwalks-orange) ![Batch](https://img.shields.io/badge/Batch-B084%20Week%204%20Capstone-orange) ![Mentor](https://img.shields.io/badge/Mentor-Waqas%20Karim%2C%20CCIE-red) ![Classification](https://img.shields.io/badge/Classification-Confidential-lightgrey)

---

##  Overview

This repository documents a **black-box penetration test** performed against a deliberately vulnerable training target, `medirozahospital.com`, as the **Week 4 Capstone Project** for the Networkwalks B084 Cybersecurity Internship. The engagement was conducted with written client authorization under a controlled, educational environment.

The assessment uncovered **7 vulnerabilities (Medium → Critical)** and chains them into a single narrative: a login page information leak escalates into a full SQL injection authentication bypass, which exposes patient medical PDFs, which — through weak passwords and leftover metadata — leads to a forgotten database backup containing every staff member's salary and the hospital's full shareholder register.

>  **Educational use only.** This report was produced in a controlled training environment with explicit written authorization. These techniques must never be used against any system without permission from its owner.

---

## Lab Environment

| Component | Detail |
|---|---|
| **Attacking OS** | Kali Linux v2026.2 (VirtualBox) |
| **Target** | `https://medirozahospital.com` (web application) |
| **Methodology** | Black-box, 4 phases — Recon → Vulnerability ID → Exploitation → Documentation |
| **Tools** | curl, browser DevTools, Networkwalks Hash Calculator, Networkwalks Password Cracker, qpdf, exiftool, wget, ChatGPT (data formatting) |
| **Scope exclusions** | Social engineering, denial-of-service, anything outside the agreed domain |

---

##  Full Attack Chain

```mermaid
flowchart TD
    A[robots.txt recon<br/>exposes /patient /staff /old] --> B[Username enumeration<br/>confirms 'admin' is valid]
     [SQL injection in login form<br/>admin' -- bypasses auth]
    [Directory listing of /patient/<br/>confirms app files]
     [3 encrypted patient PDFs<br/>downloaded from portal]
     [Weak PDF passwords<br/>cracked via wordlist attack]
    [exiftool metadata leak<br/>IT admin comment: 'DB backup moved to /old']
     [/old/ directory listing enabled<br/>mediroza_db_backup_2019.sql exposed]
     [CRITICAL: Plaintext staff salaries<br/>+ full shareholder register]
```

---

##  Findings Summary

| # | Vulnerability | Location | Risk |
|---|---|---|---|
| 1 | Username enumeration on login page | `patient/login.php` | 🟡 Medium |
| 2 | SQL injection login bypass | `patient/login.php` | 🔴 Critical |
| 3 | Encrypted PDFs accessible after login bypass | `patient/reports/` | 🟠 High |
| 4 | Weak PDF passwords crackable with a wordlist | `patient_report_*.pdf` | 🟠 High |
| 5 | Sensitive metadata left in patient PDF files | `patient_report_3.pdf` | 🟡 Medium |
| 6 | Forgotten backup folder with directory listing enabled | `old/` | 🔴 Critical |
| 7 | Confidential staff salaries & shareholder data in plain text | `old/mediroza_db_backup_2019.sql` | 🔴 Critical |

---

## 🔍 Step-by-Step Findings & Evidence

###  Recon — robots.txt Disclosure
`robots.txt` disallowed `/patient/`, `/staff/`, and `/old/` — effectively handing an attacker a map of the hidden directories it was trying to hide.

<img src="05_p4_robots-txt-curl-output.png" width="800"/>

---

###  Finding 1 — Username Enumeration (🟡 Medium)
Different error messages for a wrong username (`Username not found`) versus a wrong password (`Incorrect password`) confirmed `admin` as a valid account.

<img src="12_p9_patient-portal-login-page.png" width="800"/>

---

###  Finding 2 — SQL Injection Login Bypass (🔴 Critical)
A single quote in the username field broke the backend SQL query and returned a MySQL syntax error, confirming the field was injectable:

```sql
SELECT * FROM users WHERE username='admin'' AND password='password123'
```

The classic bypass payload authenticated as `admin` without any valid password:

```
Username: admin' --
Password: anything
```

<img src="sql_injection.jpeg" width="700"/>

<img src="13_p9_sql-error-admin-quote-dash-payload.png" width="800"/>

Once authenticated, a directory listing on `/patient/` further exposed the application's server-side files:

<img src="14_p10_directory-listing-patient-folder.png" width="800"/>

---

###  Finding 3 — Confidential PDFs Exposed (🟠 High)
The bypassed portal served three downloadable, password-protected lab report PDFs.

<img src="01_p2_patient-portal-lab-reports-list.png" width="800"/>
<img src="16_p12_patient-pdfs-unlocked-folder.png" width="800"/>
<img src="17_p13_PDF1-pathology-report-Dlamini.png" width="800"/>

**Decrypted report contents:**

| Report | Patient |
|---|---|
| `patient_report_1.pdf` | Sipho Dlamini |
| `patient_report_2.pdf` | Priya Reddy |
| `patient_report_3.pdf` | Emily Thompson |

<img src="18_p13_PDF2-pathology-report-Reddy.png" width="700"/>
<img src="19_p14_PDF3-pathology-report-Thompson.png" width="700"/>
<img src="shareholders.jpeg" width="700"/>

---

###  Finding 4 — Weak PDF Passwords (🟠 High)
All three PDFs cracked trivially using small wordlists:

| File | Cracked Password | Wordlist |
|---|---|---|
| `patient_report_1.pdf` | `123456` | Built-in 100-word list |
| `patient_report_2.pdf` | `password` | Built-in 100-word list |
| `patient_report_3.pdf` | `!@#$%^&` | Extended (JTR default) |

<img src="06_p5_password-cracker-result-password.png" width="800"/>
<img src="07_p5_password-cracker-result-123456.png" width="800"/>
<img src="08_p6_password-cracker-result-report3-special-chars.png" width="800"/>

---

###  Finding 5 — Sensitive Metadata in PDFs (🟡 Medium)
`exiftool` revealed hidden metadata in all three reports. Report 3 contained an internal IT comment pointing directly at a server backup location.

<img src="screenshots/09-exiftool-report1.jpg" width="800"/>
<img src="screenshots/10-exiftool-report2.jpg" width="800"/>

**Key leak:**
```
Author:   j.malik
Comments: DB backup moved to /old before site migration, do not delete
```

<img src="screenshots/11-exiftool-report3-itadmin-comment.jpg" width="800"/>

---

###  Finding 6 — Forgotten Backup, Directory Listing Enabled (🔴 Critical)
Following the metadata lead, `/old/` had directory listing enabled, exposing `mediroza_db_backup_2019.sql` directly to the browser.

<img src="screenshots/02-old-folder-backup-listing.jpg" width="800"/>

---

###  Finding 7 — Staff Salaries & Shareholder Data Leaked (🔴 Critical)
The backup contained **plaintext** names, roles, emails, phone numbers, national ID numbers, and salaries for all 30 staff — plus the full shareholding breakdown for all 10 hospital shareholders. Cross-referencing confirmed `j.malik` (the metadata author) is Jameel Malik, the IT Systems Administrator — tying the entire attack chain together.

<img src="screenshots/03-staff-records-leaked.png" width="800"/>
<img src="screenshots/04-staff-shareholders-table-end.jpg" width="800"/>

**Reconstructed data:**

<img src="screenshots/21-shareholders-table-reconstructed.png" width="700"/>
<img src="screenshots/22-staff-table-reconstructed.png" width="700"/>

---

##  Recommendations

| Finding | Fix |
|---|---|
| Username enumeration | Return one generic error ("Invalid credentials") for all failed logins |
| SQL injection | Use parameterized queries / prepared statements (PDO example below) |
| Exposed PDFs | Serve files outside the web root via an authenticated script; enforce 12+ char passwords |
| Metadata leakage | Strip all metadata before distributing files: `exiftool -all= file.pdf` |
| Directory listing | Add `Options -Indexes` in Apache/.htaccess; remove backups from the web root entirely |

```php
// Safe login query — PHP PDO prepared statement
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ? AND password = ?");
$stmt->execute([$username, $password]);
```

---

##  Conclusion

A complete attack chain — from a single disclosed directory to full financial and ownership data exposure — was achieved using nothing but publicly documented, well-known techniques. All **Critical** and **High** findings should be remediated before this system handles real patient data.

---

##  Credits

| Role | Name |
|---|---|
| **Submitted by** | Usoro, Iniubong Etido |
| **Cybersecurity Mentor** | Waqas Karim, CCIE |
| **Organization** | Networkwalks |
| **Program** | Batch B084 — Week 4 Capstone Project |

> This report was produced as part of a controlled educational exercise by Networkwalks. The target was authorized for security testing. These techniques must never be applied to any system without explicit written permission from the owner.
