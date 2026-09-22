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
