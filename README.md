# Kobold HTB - Write-up & Privilege Escalation Guide

## 📝 Brief Description
Kobold is a medium-difficulty Hack The Box machine that tests advanced pivoting and container exploitation techniques. The main attack chain involves leveraging known vulnerabilities combined with misconfigured system group permissions and locally cached Docker environments.

---

## 🔍 Reconnaissance & Initial Access

### 1. API Mapping and SSRF
* During the web enumeration phase, an exposed administration panel for the **Arcane** application was identified running on port `3552`.
* Analyzing the API endpoints revealed a **Server-Side Request Forgery (SSRF)** vulnerability within the `?url=` parameter of the `/api/templates/fetch` endpoint.
* Since the local API implicitly trusted internal requests, the SSRF was used to map protected endpoints (such as `/api/environments/0/settings/public`), successfully compromising initial credentials.

### 2. Foothold
* Utilizing the retrieved credentials and exploiting a **Remote Code Execution (RCE)** flaw in the **MCPJam** platform (`https://mcp.kobold.htb/api/mcp/connect`), a reverse shell was established back to our attacking machine (Kali Linux).
* This granted initial foothold on the target host as the user **`ben`** (UID 1001).

---

## 🚀 Privilege Escalation

### 1. Local Reconnaissance
Once inside the system as `ben`, standard enumeration commands were executed to map out the privilege escalation vector:

```bash
ben@kobold:~$ id
uid=1001(ben) gid=1001(ben) groups=1001(ben),37(operator)
