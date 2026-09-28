# AD Home Lab - Kerberoasting Attack & Detection

A self-built Active Directory environment used to practice core AD administration and then attack is as a red teamer would. This is followed by identifying the attacker in the Domain Controller's event logs. Built entirely in VirtualBox on an isolated host-only network.

This project was to help with hands-on Active Directory learning. It is to demonstrate what an easy-to-make, real life misconfiguration (such as an over-privileged service account with a very weak password) looks like from both sides. The attacker (red team) exploiting it, and the defender (blue team) identifying it.

## Environment

| Machine | Role | IP | OS |
|---|---|---|---|
| DC01 | Domain Controller | 192.168.56.10 | Windows Server 2022
(Desktop Experience) |
| CLIENT01 | Domain-joined workstation | 192.168.56.11 | Windows 11
Enterprise Evaluation |
| Kali | Attacker box | 192.168.56.103 | Kali Linux |

All three VMs sit on a VirtualBox **host-only network**
('192.168.56.0/24'), isolated from the internet and rest of the machine. This helps with ensuring the lab is self-contained and safe to attack without any external risk.

Domain: 'corp.local' (NetBIOS: 'CORP')

## Table of Contents

1. [VM & Network Setup](#1-vm--network-setup)
2. [Domain Controller Promotion](#2-domain-controller-promotion)
3. [Domain Join](#3-domain-join)
4. [Users, Group & OUs](#4-users-groups--ous)
5. [File Share & Group Policy](#5-file-share--group-policy)
6. [Sysmon Deployment](#6-sysmon-deployment)
7. [Attack: Kerberoasting](#7-attack-kerberoasting)
8. [Detection: Verifying the Attack in Event Logs](#8-detection-verifying-the-attack-in-event-logs)
9. [What I'd Do Next](#9-what-id-do-next)

---

## 1. VM & Network Setup

Before anything else, I created and fully set up an isolated host-only network in VirtualBox ('192.168.56.0/24') so that all three VMs were able to communicate with each other without having to interfere with my host machine's network or internet.

![Host-only network](vm-setup/hostonly-network-created.png)

DC01: Windows Server 2022 Standard Evaluation (Desktop Experience), 6GB RAM, 2 CPUs, 60GB disk. This was built first.

![DC01 VM creation](vm-setup/dc01-vm-creation-name-edition.png)

CLIENT01: Windows 11 Enterprise Evaluation. This was built next. While making this virtual machine, I hit a disk size error as I accidentally undersized the install disk and therefore Windows Setup refused to install. ("The system drive needs to be at least 52GB or larger"). Seeing this, I went back into VirtualBox and resized the virtual disk to 64 GB. I rebooted the machine and the setup ran successfully.

![Disk too small error](vm-setup/client01-disk-error-52gb-warning.png)
![Disk resized to 64GB](vm-setup/client01-disk-resized-64gb.png)

The DC01 machine also attempted to boot before the ISO mounted properly, causing it to give me a boot-order error ('0xc0000225'). This was resolved by checking and correcting the boot device order in the VM settings.

## 2. Domain Controller Promotion

With DC01 built and set up, I installed the **Active Directory Domain Services** role and promoted the server to stand up a new forest and domain: 'corp.local'.

![Selecting the AD DS role](dc-promotion/dc01-addsrole-checkbox.png)

During promotion, I configured the domain controller options (forest/domain functional level, DNS server, Global Catalog) and set the Directory Services Restore Mode recovery password.

![Domain Controller Options](dc-promotion/dc01-domain-controller-options.png)

After the requirement check had passed and rebooted, I made sure the promotion went through by logging in as 'CORP\Administrator' and examining the network configuration with 'ipconfig /all'.

![Logged in as CORP\Administrator](dc-promotion/dc01-corp-administrator-login.png)
![ipconfig verifying domain identity](dc-promotion/dc01-ipconfig-verify.png)

## 3. Domain Join

Joining CLIENT01 to the domain caused me a substantial amount of issues, which I find value in. It's a more realistic picture of what setting up AD actually looks like.

First, a static IP was set and pointed CLIENT01's DNS at DC01 ('192.168.56.10'). The DC must be used as the domain-joined machine's DNS server to resolve the domain.

![Static IP and DNS config](domain-join/client01-static-ip-config.png)

When attempting a ping session to DC01 it failed. After investigating, it turns out DC01's firewall wasn't allowing inbound ICMP by default. To resolve this, I checked and enabled the relevant inbound firewall rule on DC01.

![Ping failing](domain-join/client01-ping-dc01-failed.png)
![DC01 firewall ICMP rule](domain-join/dc01-firewall-icmp-rule-check.png)

After fixing that, ping and 'nslookup corp.local' both succeeded, confirming name resolution was working correctly before attempting the join.

![Ping succeeding](domain-join/client01-ping-dc01-success.png)
![nslookup resolving corp.local](domain-join/client01-nslookup-corp-local.png)

I then joined CLIENT01 to 'corp.local'. The machine's name was set to default ('DESKTOP-BLBDE7L'), so I renamed it to 'CLIENT01' using 'Rename-Computer' and verified the new hostname after a restart.

![Welcome to the domain](domain-join/client01-domain-join-welcome.png)
![Rename-Computer to CLIENT01](domain-join/client01-hostname-before.png)
![Confirmed hostname CLIENT01](domain-join/client01-hostname-after.png)

Verified from DC01 too. CLIENT01 pops up as a computer object in Active Directory Users and Computers.

![CLIENT01 computer object in ADUC](domain-join/dc01-aduc-client01-computer-object.png)

## 4. Users, Groups & OUs

I organized the domain with a basic OU structure ('Employees', 'Workstations', 'Groups'), created a domain user ('jsmith' - John Smith), and created and configured a security group ('FileShare-Users') to control access to a shared folder.

![OU structure](users-and-groups/dc01-aduc-ou-structure.png)
![New user jsmith](users-and-groups/dc01-aduc-new-user-jsmith.png)
![FileShare-Users group membership](users-and-groups/dc01-aduc-group-membership.png)

## 5. File Share & Group Policy

I prepared a shared folder ('\\DC01\Company') with NTFS permissions scoped to the 'FileShare-Users' group, and automatically mapped it as a drive letter on domain-joined machines using Group Policy. This way, users don't have to manually go through and connect to the share.

Two additional GPOs were configured to harden the environment:
- A **Control Panel restriction** policy to limit what standard users can change on their machines'
- A **domain password and account lockout policy** (in Default Domain Policy). This will be useful later, since it is what triggers the account lockout during the soon-to-come password spray.

## 6. Sysmon Deployment

I installed **Sysmon** before running any attacks (using the SwiftOnSecurity community configuration) on both DC01 and CLIENT01. Sysmon logs much more security-relevant detail than the default auditing by Windows. Process creation with full command lines, parent/child process relationships, and more. This is very important as the detection side of this project will rely on Sysmon.

![Sysmon install on DC01](sysmon/dc01-sysmon-install.png)
![Sysmon install on CLIENT01](sysmon/client01-sysmon-install.png)

I made sure Sysmon was capturing events by looking at Event Viewer for a live Event ID 1 (process creation) on CLIENT01.

![Sysmon Event ID 1 in Event Viewer](sysmon/client01-sysmon-event-1-process-create.png)

## 7. Attack: Kerberoasting

After finishing the entire environment, I switched to the Kali VM and ran a **Kerberoasting** attack chain against a vulnerable service account. This is a realistic simulation of how an attacker could escalate from a low-privilege standing to a crackable credential.

**Setup:** I created a false service account, 'svc_sql', and registered a Service Principal Name on it ('MSSQLSvc/dc01.corp.local:1433'). This simulates a very common real Kerberoasting target, a SQL Server service account. Most importantly though, I gave it a weak password to make it easy to crack.

![SPN registered on svc_sql](kali-attacks/dc01-svc-sql-spn-set.png)

**Recon:** From Kali, I ran an 'nmap' scan against DC01 to verify connection and identify open services.

![nmap scan of DC01](kali-attacks/kali-nmap-dc01.png)

**Initial access - password spray:** Using 'netexec', I tried to log into DC01 as 'jsmith' with a list of guessable passwords. This forced the account to enter its lockout mode, simulating a noisy but real outcome of a spray attack.

![netexec password spray triggering lockout](kali-attacks/kali-netexec-spray-lockout.png)

**Kerberoasting:** I requested a Kerberos service ticket for the 'svc_sql' account using Impacket's 'GetUserSPNs'. Due to that account having an SPN set, any authenticated domain user can request a ticket for it. The ticket is also encrypted with a hash of the service account's own password. Because of this, it can be cracked offline, never having to touch the DC again.

![GetUserSPNs pulling the ticket hash](kali-attacks/kali-getuserspns-ticket.png)

**Cracking:** My first attempt used 'hashcat', but there was no GPU passthrough available with my Kali VM, making GPU-accelerated cracking impossible in this environment. Instead, I decided to turn towards **John the Ripper** with a custom-made wordlist instead, cracking the hash instantly.

![hashcat failing due to no GPU](kali-attacks/kali-hashcat-no-gpu-error.png)
![John the Ripper cracking svc_sql's password](kali-attacks/kali-john-cracked-svc-sql.png)

The cracked password was 'homelab2026!'. This proves that a weak service account password mixed with an SPN is plenty for a domain user to compromise the account's credentials even while offline.

## 8. Detection: Verifying the Attack in Event Logs

The next half of this project, and what I personally found most valuable, was returning to DC01 and verifying that every single step of the attack had left a paper trail in the Windows Security event log.

**Failed logon (the password spray):** Event ID **4625** on DC01 expresses the failed 'jsmith' logon attempt, also including the Kali VM's source IP ('192.168.56.103'). A SOC analyst or SIEM alert rule would flag this exact thing.

![Event 4625 — failed logon from Kali's IP](kali-attacks/dc01-event-4625-network-source.png)

**Kerberos ticket request (the actual Kerberoasting):** Event ID **4679** expresses the Kerberos service ticket request for 'svc_sql'. A defender, upon seeing this event, would monitor to catch the Kerberoasting in progress, especially when connected to a spike in ticket requests for accounts with SPNs.

![Event 4769 — Kerberos ticket requested for svc_sql](kali-attacks/dc01-event-4769-ticket-request.png)

This loop of misconfiguring, attacking, and confirming visiblity in logs, best represents the purple-team mindset in cybersecurity. You must understand and master an attack process to both execute and detect it.

## 9. What I'd Do Next

I would like to explore many things if I continue with this lab:
- Forward Sysmon and Security event logs to a SIEM (e.g. Splunk or a free ELK stack) to create actual detection rules and alerts instead of having to manually go through Event Viewer and find the logs
- Add another less obvious attack path (e.g. AS-REP roasting, or abusing a misconfigured GPO) to widen the detection coverage
- Script the entire environment with PowerShell DSC or Ansible so that the lab can be fully reproduced from scratch.
