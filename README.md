# 🛡️ SentryX – Secure Virtual Infrastructure Lab 🚀

![Arch Linux](https://img.shields.io/badge/Arch%20Linux-rolling-blue?logo=archlinux) ![KVM/QEMU](https://img.shields.io/badge/KVM%2FQEMU-virtualization-333?logo=qemu) ![libvirt](https://img.shields.io/badge/libvirt-enabled-4c9) ![pfSense](https://img.shields.io/badge/pfSense-firewall-1f4a7f?logo=pfsense) ![Windows Server 2025](https://img.shields.io/badge/Windows%20Server-2025-0078d6?logo=windows) ![Debian](https://img.shields.io/badge/Debian-GLPI%2FZabbix%2FWazuh-a80030?logo=debian) ![TrueNAS](https://img.shields.io/badge/TrueNAS-CORE-0b6aa2?logo=truenas) ![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

---

## 🇺🇸 English — Project Overview

SentryX is a **virtualized cybersecurity lab** on a single Arch Linux host using **KVM/QEMU + libvirt**. It reproduces a **practical enterprise environment** with a firewall, an AD domain, service servers (GLPI, Zabbix, Wazuh), a NAS (TrueNAS), and a client network — all segmented by VLANs with strict rules.

### 🎯 Goals

* Build a **segmented, secure, and reproducible** lab.
* Operate **enterprise-grade services**: AD/DNS, GLPI, Zabbix, Wazuh, TrueNAS.
* Apply **best practices** (VLANs, hardening, centralized logging & monitoring).
* Demonstrate **AIS REAC coverage** (CP1–CP10) through concrete tasks.
* Provide **clear documentation** and a coherent **addressing plan**.

---

## 🧩 Host Platform (Hypervisor)

* **Host OS:** Arch Linux (rolling)
* **Virtualization:** KVM/QEMU, libvirt (virt-manager for UI)
* **vNICs:** virtio (paravirtualized)
* **vDisks:** virtio, `qcow2` images (SSD-backed)
* **Bridging:** Linux bridge for VLAN trunk to pfSense, per-VLAN attachment for VMs

---

## 🖥️ Virtual Machines

| Virtual Machine     | OS (Version)               | vCPUs |  RAM  |  Disk | Purpose                                     |
| ------------------- | -------------------------- | :---: | :---: | :---: | ------------------------------------------- |
| pfSense (Firewall)  | pfSense CE 2.8.1 (FreeBSD) |   2   |  4 GB | 20 GB | Perimeter FW, inter-VLAN routing, DHCP, NTP |
| Windows Server (DC) | Windows Server 2025 (24H2) |   4   |  8 GB | 60 GB | AD DS, DNS, GPO, domain time source         |
| GLPI                | Debian 13 + GLPI 10.0.19   |   2   |  4 GB | 20 GB | Helpdesk, inventory agent, AD auth          |
| Zabbix              | Debian 13 + Zabbix 7.0 LTS |   4   |  8 GB | 20 GB | Monitoring (agents on all VMs)              |
| Wazuh               | Debian 13 + Wazuh 4.12.0   |   4   |  8 GB | 20 GB | SIEM/XDR: log collection, rules, alerts     |
| TrueNAS CORE (NAS)  | TrueNAS CORE 13 (FreeBSD)  |   4   | 16 GB | 60 GB | ZFS storage, SMB/NFS shares, snapshots      |

> Allocation exceeds vendor minimums to keep the lab responsive under load.

---

## 🌐 Network Design

### VLANs & Addressing

| VLAN | Name              | Subnet/CIDR   | Gateway    | DHCP Pool                 | Key Hosts (Static)                                   |
| :--: | ----------------- | ------------- | ---------- | ------------------------- | ---------------------------------------------------- |
|  10  | Infrastructure    | 10.10.10.0/24 | 10.10.10.1 | 10.10.10.100–10.10.10.199 | pfSense: 10.10.10.1 · DC (AD/DNS): 10.10.10.10       |
|  20  | Services          | 10.20.20.0/24 | 10.20.20.1 | 10.20.20.100–10.20.20.199 | GLPI: 10.20.20.20 · Zabbix: 10.20.20.30 · Wazuh: .40 |
|  30  | Clients & Storage | 10.30.30.0/24 | 10.30.30.1 | 10.30.30.100–10.30.30.199 | TrueNAS: 10.30.30.10 · Win Client: 10.30.30.100      |

**Domain & DNS**

* **AD Domain:** `sentryx.lab`
* **DNS (authoritative):** 10.10.10.10 (DC); pfSense forwards upstream.
* **DHCP Options:** Router (VLAN GW), DNS = 10.10.10.10, Domain = `sentryx.lab`

**NTP**

* pfSense serves NTP (upstream pool.ntp.org).
* Domain members sync via the DC (Windows hierarchy).

**Routing & NAT**

* Inter-VLAN routing on pfSense.
* Outbound NAT (automatic) for Internet access.

**Firewall Policy (summary)**

* VLAN 10 → others: management (RDP/SSH/HTTPS) + AD/DNS.
* VLAN 20 → VLAN 10: AD/DNS/LDAP/Kerberos only; block unsolicited inbound.
* VLAN 30 → VLAN 10/20: domain join, GLPI (HTTPS), monitoring agents; deny admin ports.

**Service Ports (reference)**

* **AD/DC:** DNS 53 TCP/UDP · LDAP 389 TCP · Kerberos 88 TCP/UDP · SMB 445 TCP · RDP 3389 TCP
* **GLPI:** 443 TCP
* **Zabbix:** 10051 TCP (server), 10050 TCP (agents)
* **Wazuh:** 1514 UDP (logs), 55000 TCP (API/registration)
* **TrueNAS:** SMB 445 TCP · NFS 2049 TCP (+ dynamic ports)

---

## 🔐 Identity & Access (AD)

**OU Structure (example)**

```
OU=Admins
OU=Servers
OU=Workstations
OU=ServiceAccounts
OU=Groups
```

**Key Groups**

* `GRP-Admins-Domain`
* `GRP-Zabbix-Agents`
* `GRP-Wazuh-Agents`
* `GRP-GLPI-Users`

**GPO Highlights**

* Baseline hardening (password policy, lock screen, SMB signing).
* NTP per domain hierarchy.
* Windows Firewall with rules aligned to lab ports.
* DNS client = 10.10.10.10.

---

## 🗃️ Storage Design (TrueNAS / ZFS)

**Pool:** `tank`
**Datasets & shares**

* `tank/shares/it` → SMB `\\truenas\it`
* `tank/backups` → SMB `\\truenas\backups`
* `tank/homes` → SMB home directories (optional)

**Snapshots**

* Hourly (24), Daily (7), Weekly (4).

**Permissions**

* SMB with ACLs; admin shares limited to VLAN 10.

---

## 🛠️ Services Configuration

**GLPI** — `https://glpi.sentryx.lab`

* AD LDAP bind to 10.10.10.10, sync users/groups.
* GLPI Agent for asset/software inventory.

**Zabbix** — `https://zabbix.sentryx.lab`

* Agents on Windows/Debian/FreeBSD hosts.
* Templates: OS, FS, NIC, CPU/RAM.
* Triggers: high CPU, low disk, agent unreachable.

**Wazuh** — `https://wazuh.sentryx.lab`

* Agents on all VMs; watch Windows Events, auth, sudo, SSH.
* Rules: failed auth thresholds, privilege escalation, suspicious processes.

---

## 🔍 Monitoring & SIEM — Data Flow

* **Windows Server →** Zabbix agent + Wazuh agent (Event Logs).
* **Debian servers →** Zabbix agent + Wazuh agent (syslog/auth).
* **pfSense →** Zabbix (SNMP/agent) + syslog to Wazuh.
* **TrueNAS →** Zabbix (agent/SNMP) + syslog to Wazuh.

---

## 🧪 Incident Scenario (sample)

Repeated failed logins on a Windows client (VLAN 30) followed by a successful login from an unusual source.
**Expected outcome:** Wazuh brute-force alert, Zabbix event spikes, admin validates source IP and locks the account or resets password in AD.

---

## 🔧 pfSense & libvirt Mapping

* **pfSense NICs:** `WAN` (bridged to uplink), `LAN-TRUNK` (virtio on Linux bridge, tagged VLANs 10/20/30)
* **pfSense VLAN IFs:** VLAN10 = 10.10.10.1/24 · VLAN20 = 10.20.20.1/24 · VLAN30 = 10.30.30.1/24
* **VM NICs:** attached to their target VLAN (tagged or per-VLAN bridge)

---

## 📊 REAC (AIS) Mapping

| REAC (AT/CP)       | SentryX Implementation                                                                     |
| ------------------ | ------------------------------------------------------------------------------------------ |
| **AT1 (CP1–CP4)**  | pfSense VLANs & firewall, OS updates, hardening, Wazuh agents, ZFS ACLs                    |
| **AT2 (CP5–CP7)**  | Deploy AD/DNS, GLPI (AD auth), Zabbix; design VLANs; integrate TrueNAS backups             |
| **AT3 (CP8–CP10)** | Centralized monitoring (Zabbix), SIEM (Wazuh), incident analysis & response, documentation |

---

## 🔗 Service Endpoints

* pfSense: `https://10.10.10.1`
* DC (AD/DNS): `dc1.sentryx.lab`
* GLPI: `https://glpi.sentryx.lab`
* Zabbix: `https://zabbix.sentryx.lab`
* Wazuh: `https://wazuh.sentryx.lab`
* TrueNAS: `https://truenas.sentryx.lab`

---

## 📄 License (MIT)

Copyright © 2025 **ZTr1∂n R.J.**
This project is licensed under the **MIT License**. See the [Licence.md](https://github.com/0x1void/SentryX/blob/main/Licence.md) file for details.

---

## 🤝 Contributing

Contributions are welcome. See the [CONTRIBUTING.md](https://github.com/0x1void/SentryX/blob/main/CONTRIBUTING.md) file for details.

* **Issues:** Open an issue describing the change or problem.
* **Fork & Branch:** `feat/…` or `fix/…` branch naming.
* **Commits:** Clear, concise messages.
* **PRs:** One topic per PR, with a short rationale and test notes.
  By contributing, you agree your work will be licensed under the project’s **MIT License**.

---

✍️ *Développé par ZTr1∂n R.J. – 2025*

---

# 🇫🇷 Français — Aperçu du projet

SentryX est un **laboratoire de cybersécurité virtualisé** sur une machine Arch Linux avec **KVM/QEMU + libvirt**. Il reproduit un **environnement d’entreprise concret** : pare-feu, domaine AD, serveurs de services (GLPI, Zabbix, Wazuh), NAS (TrueNAS) et réseau clients — le tout segmenté en VLANs avec des règles strictes.

### 🎯 Objectifs

* Construire un lab **segmenté, sécurisé et reproductible**.
* Exploiter des **services professionnels** : AD/DNS, GLPI, Zabbix, Wazuh, TrueNAS.
* Appliquer les **bonnes pratiques** (VLANs, durcissement, logs & supervision centralisés).
* Montrer la **couverture REAC AIS** (CP1–CP10) via des tâches concrètes.
* Fournir une **documentation claire** et un **plan d’adressage** cohérent.

---

## 🧩 Plateforme hôte (Hyperviseur)

* **OS hôte :** Arch Linux (rolling)
* **Virtualisation :** KVM/QEMU, libvirt (virt-manager)
* **vNICs :** virtio (paravirtualisées)
* **vDisks :** virtio, images `qcow2` (SSD)
* **Bridging :** pont Linux pour le trunk VLAN vers pfSense, attachement par VLAN pour les VMs

---

## 🖥️ Machines virtuelles

| Machine virtuelle   | OS (Version)               | vCPU |  RAM  | Disque | Rôle                                         |
| ------------------- | -------------------------- | :--: | :---: | :----: | -------------------------------------------- |
| pfSense (Pare-feu)  | pfSense CE 2.8.1 (FreeBSD) |   2  |  4 Go |  20 Go | Pare-feu, routage inter-VLAN, DHCP, NTP      |
| Windows Server (DC) | Windows Server 2025 (24H2) |   4  |  8 Go |  60 Go | AD DS, DNS, GPO, source de temps du domaine  |
| GLPI                | Debian 13 + GLPI 10.0.19   |   2  |  4 Go |  20 Go | Helpdesk, inventaire, auth AD                |
| Zabbix              | Debian 13 + Zabbix 7.0 LTS |   4  |  8 Go |  20 Go | Supervision (agents sur toutes les VMs)      |
| Wazuh               | Debian 13 + Wazuh 4.12.0   |   4  |  8 Go |  20 Go | SIEM/XDR : collecte de logs, règles, alertes |
| TrueNAS CORE (NAS)  | TrueNAS CORE 13 (FreeBSD)  |   4  | 16 Go |  60 Go | Stockage ZFS, partages SMB/NFS, snapshots    |

---

## 🌐 Conception réseau

### VLANs & adressage

| VLAN | Nom                | Sous-réseau   | Passerelle | Plage DHCP                | Hôtes clés (statique)                                   |
| :--: | ------------------ | ------------- | ---------- | ------------------------- | ------------------------------------------------------- |
|  10  | Infrastructure     | 10.10.10.0/24 | 10.10.10.1 | 10.10.10.100–10.10.10.199 | pfSense : 10.10.10.1 · DC (AD/DNS) : 10.10.10.10        |
|  20  | Services           | 10.20.20.0/24 | 10.20.20.1 | 10.20.20.100–10.20.20.199 | GLPI : 10.20.20.20 · Zabbix : 10.20.20.30 · Wazuh : .40 |
|  30  | Clients & Stockage | 10.30.30.0/24 | 10.30.30.1 | 10.30.30.100–10.30.30.199 | TrueNAS : 10.30.30.10 · Client Windows : 10.30.30.100   |

**Domaine & DNS**

* **Domaine AD :** `sentryx.lab`
* **DNS (autoritatif) :** 10.10.10.10 (DC) ; pfSense redirige en amont.
* **Options DHCP :** Passerelle (VLAN), DNS = 10.10.10.10, Domaine = `sentryx.lab`

**NTP**

* pfSense sert de NTP (amont : pool.ntp.org).
* Les membres du domaine se synchronisent via le DC.

**Routage & NAT**

* Routage inter-VLAN via pfSense.
* NAT sortant automatique vers Internet.

**Règles de pare-feu (synthèse)**

* VLAN 10 → autres : administration (RDP/SSH/HTTPS) + AD/DNS.
* VLAN 20 → VLAN 10 : AD/DNS/LDAP/Kerberos uniquement ; pas d’entrant non sollicité.
* VLAN 30 → VLAN 10/20 : jonction au domaine, GLPI (HTTPS), agents de supervision ; pas de ports d’admin.

**Ports (référence)**

* **AD/DC :** DNS 53 TCP/UDP · LDAP 389 TCP · Kerberos 88 TCP/UDP · SMB 445 TCP · RDP 3389 TCP
* **GLPI :** 443 TCP
* **Zabbix :** 10051 TCP (serveur), 10050 TCP (agents)
* **Wazuh :** 1514 UDP (logs), 55000 TCP (API/enrôlement)
* **TrueNAS :** SMB 445 TCP · NFS 2049 TCP (+ dynamiques)

---

## 🔐 Identité & accès (AD)

**Structure d’OU (exemple)**

```
OU=Admins
OU=Servers
OU=Workstations
OU=ServiceAccounts
OU=Groups
```

**Groupes clés**

* `GRP-Admins-Domain`
* `GRP-Zabbix-Agents`
* `GRP-Wazuh-Agents`
* `GRP-GLPI-Users`

**Points GPO**

* Durcissement de base (politique de mot de passe, verrouillage, SMB signing).
* NTP conforme à la hiérarchie de domaine.
* Pare-feu Windows actif avec règles alignées sur les ports du lab.
* DNS client = 10.10.10.10.

---

## 🗃️ Stockage (TrueNAS / ZFS)

**Pool :** `tank`
**Datasets & partages**

* `tank/shares/it` → SMB `\\truenas\it`
* `tank/backups` → SMB `\\truenas\backups`
* `tank/homes` → répertoires personnels (option)

**Snapshots**

* Horaire (24), quotidien (7), hebdomadaire (4).

**Permissions**

* ACLs fines ; partages admin limités au VLAN 10.

---

## 🛠️ Configuration des services

**GLPI** — `https://glpi.sentryx.lab`

* Liaison LDAP vers 10.10.10.10, synchro utilisateurs/groupes.
* Agent GLPI pour inventaire matériel/logiciel.

**Zabbix** — `https://zabbix.sentryx.lab`

* Agents Windows/Debian/FreeBSD.
* Templates : OS, systèmes de fichiers, interfaces réseau, CPU/RAM.
* Déclencheurs : CPU élevé, disque bas, agent injoignable.

**Wazuh** — `https://wazuh.sentryx.lab`

* Agents sur toutes les VMs ; suivi : Event Logs Windows, auth, sudo, SSH.
* Règles : échecs d’authentification, élévation de privilèges, processus suspects.

---

## 🔍 Supervision & SIEM — Flux

* **Windows Server →** agent Zabbix + agent Wazuh (Event Logs).
* **Serveurs Debian →** agent Zabbix + agent Wazuh (syslog/auth).
* **pfSense →** Zabbix (SNMP/agent) + syslog vers Wazuh.
* **TrueNAS →** Zabbix (agent/SNMP) + syslog vers Wazuh.

---

## 🧪 Scénario d’incident (exemple)

Multiples échecs de connexion sur un poste Windows (VLAN 30) suivis d’une réussite depuis une source inhabituelle.
**Résultat attendu :** alerte Wazuh, pic d’événements Zabbix, vérification de l’IP source, verrouillage du compte ou réinitialisation du mot de passe via l’AD.

---

## 🔧 Cartographie pfSense & libvirt

* **Interfaces pfSense :** `WAN` (pont vers l’uplink), `LAN-TRUNK` (virtio sur pont Linux, VLANs 10/20/30 taggés)
* **VLAN pfSense :** VLAN10 = 10.10.10.1/24 · VLAN20 = 10.20.20.1/24 · VLAN30 = 10.30.30.1/24
* **NIC des VMs :** reliées au VLAN correspondant (tag ou pont dédié)

---

## 📊 Correspondance REAC (AIS)

| REAC (AT/CP)       | Mise en œuvre SentryX                                                                          |
| ------------------ | ---------------------------------------------------------------------------------------------- |
| **AT1 (CP1–CP4)**  | VLANs & firewall pfSense, mises à jour, durcissement, agents Wazuh, ACLs ZFS                   |
| **AT2 (CP5–CP7)**  | Déploiement AD/DNS, GLPI (auth AD), Zabbix ; design VLAN ; sauvegardes TrueNAS                 |
| **AT3 (CP8–CP10)** | Supervision centralisée (Zabbix), SIEM (Wazuh), analyse & réponse aux incidents, documentation |

---

## 🔗 Points d’accès

* pfSense : `https://10.10.10.1`
* Contrôleur de domaine (AD/DNS) : `dc1.sentryx.lab`
* GLPI : `https://glpi.sentryx.lab`
* Zabbix : `https://zabbix.sentryx.lab`
* Wazuh : `https://wazuh.sentryx.lab`
* TrueNAS : `https://truenas.sentryx.lab`

---

## 📄 Licence (MIT)

Copyright © 2025 **ZTr1∂n R.J.**
Ce projet est sous licence **MIT**. Voir le fichier [Licence.md](https://github.com/0x1void/SentryX/blob/main/Licence.md) pour les détails.

---

## 🤝 Contribution

Les contributions sont les bienvenues. Voir le fichier [CONTRIBUTING.md](https://github.com/0x1void/SentryX/blob/main/CONTRIBUTING.md) file for details.

* **Issues :** décrire clairement le besoin ou le problème.
* **Fork & Branche :** nommage `feat/…` ou `fix/…`.
* **Commits :** messages courts et explicites.
* **PR :** un sujet par PR, avec un résumé et des notes de test.
  Toute contribution est publiée sous la **licence MIT** du projet.

---

✍️ *Développé par ZTr1∂n R.J. – 2025*
