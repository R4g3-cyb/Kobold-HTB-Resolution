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

Key Findings:

    The user belongs to the operator group (GID 37), a secondary group that typically possesses privileges to interact indirectly with system sub-processes.

    Checking running processes revealed the binary arcane_linux_amd64 executing with root privileges.

    The web infrastructure was running a vulnerable version of PrivateBin (v2.0.2), exposed to CVE-2025-64714 / GHSA-g2j9-g8r5-rg82 (Local File Inclusion via template-switching).

2. The Isolated Environment Dilemma

According to community advisory notes, the local Docker image used for PrivateBin (privatebin/nginx-fpm-alpine:2.0.2) unintentionally allowed mounting the host's root filesystem (/) into the container.

However, attempting to interact directly with the Docker daemon resulted in permission errors:
Bash

ben@kobold:~$ docker run ...
docker: permission denied while trying to connect to the Docker daemon socket...

3. Exploitation: Abusing sg and Bypassing Restrictions

To bypass the Docker socket restriction, the sg (execute command as different group ID) command was utilized. This allowed us to inherit the privileges of the docker group through our operator assignment.

When attempting to spin up a clean Linux image (such as alpine:latest) to perform the mount, the Docker Engine failed because the target machine lacked outbound Internet access (context deadline exceeded).

To solve this, the Docker engine was forced to reuse the vulnerable image already cached locally: privatebin/nginx-fpm-alpine:2.0.2.
The Entrypoint Block and SUID Defense

Running the PrivateBin image directly to inject SUID permissions (chmod +s /bin/bash) failed because the container ignored the trailing command by default and started its internal web server instead (fpm is running... ready to handle connections), hanging the terminal.

Furthermore, host security mechanisms (such as AppArmor/Seccomp or a nosuid mount flag) threw an Operation not permitted error when trying to modify the host's actual Bash binary.
4. The Final Blow: Arbitrary File Read as Root

To bypass all SUID restrictions and evade the default web service behavior, multiple arguments were chained into a single command line:

    sg docker -c: To execute Docker using the required secondary group privileges.

    --entrypoint /bin/sh: To override the default web server startup script and force a clean shell execution.

    -v /:/hostfs: To map the actual host root directory into the /hostfs folder inside the container.

    -u root: To force the container to run as the highest administrative user, breaking down file system permission barriers over the mounted volume.

Command Executed:
Bash

sg docker -c "docker run --rm -u root --entrypoint /bin/sh -v /:/hostfs privatebin/nginx-fpm-alpine:2.0.2 -c 'cat /hostfs/root/root.txt'"

Result: The container spawned locally and instantly, read the protected root flag through the mapped volume thanks to the -u root flag, and printed the administrative hash directly into ben's terminal session.
