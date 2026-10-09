# SIEM Log Analysis & Alert Tuning — Wazuh Home Lab Writeup

**Date:** July 2026
**Type:** Personal Home Lab Project
**Level:** Beginner → Intermediate
**Category:** Blue Team / SOC / Detection Engineering

---

## Overview

This project is a hands-on home lab built to practice the core responsibilities of a **SOC Analyst / Detection Engineer**: deploying a SIEM, forwarding logs from a monitored endpoint, generating realistic attack traffic, analyzing the resulting alerts, tuning detection rules to reduce noise, and finally automating a defensive response.

The idea originated from a TryHackMe "Project 01" prompt (*"Deploy Splunk, ELK, or Wazuh to collect and correlate logs across your environment, then tune alert thresholds to cut the noise and surface real threats faster"*). **Wazuh** was chosen as the SIEM stack because it is fully open-source, lightweight enough to run in a local VirtualBox lab, and ships with a built-in alerting/rules engine — making it the most realistic option for a single-laptop home lab.

Skills practiced:
- Virtual infrastructure design (isolated dual-adapter networking)
- Linux server administration & systemd service troubleshooting
- SIEM deployment (Wazuh Manager, Indexer, Dashboard, Agent)
- Log analysis and MITRE ATT&CK-mapped detection
- Custom detection rule authoring (XML rule correlation, `frequency`/`timeframe`)
- Offensive simulation (Hydra, manual scripted brute force) to validate detections
- Active Response automation (auto firewall blocking + auto/manual unblocking)

---

## Objective

The goal was not just to "install a SIEM," but to demonstrate the **full defensive lifecycle**: *deploy → detect → analyze → tune → automate response*. This mirrors real SOC workflows where raw detections are rarely useful out of the box — they need to be correlated and tuned against actual attack data before they become actionable, and ideally followed by automated containment.

---

## Lab Setup

### Topology

```
┌─────────────────────────── Windows 11 Host ───────────────────────────┐
│  VirtualBox Host-Only Adapter: 192.168.56.1                            │
│                                                                          │
│   ┌──────────────────┐   ┌───────────────────────┐   ┌──────────────┐ │
│   │   wazuh-server    │   │  endpoint-linux-wazuh │   │     kali     │ │
│   │  (Wazuh Manager,  │   │   (Wazuh Agent /      │   │  (Attacker / │ │
│   │  Indexer,         │◄──┤   monitored host)     │◄──┤   Hydra)     │ │
│   │  Dashboard)       │   │                       │   │              │ │
│   │  192.168.56.104   │   │   192.168.56.106      │   │ 192.168.56.102│ │
│   └──────────────────┘   └───────────────────────┘   └──────────────┘ │
│      Ubuntu Server 24.04       Ubuntu Server 24.04       Kali Linux    │
│      8GB RAM / 4 vCPU          2GB RAM / 2 vCPU           (pre-existing)│
└──────────────────────────────────────────────────────────────────────┘
```

All three VMs run in **VirtualBox** with **dual network adapters**:
- **Adapter 1 — NAT**: outbound internet access (package installs/updates)
- **Adapter 2 — Host-only Network**: isolated `192.168.56.0/24` subnet allowing VM↔VM and VM↔Host communication without exposing the lab to the internet

### Specifications

| VM | Role | OS | RAM | vCPU | Disk |
|---|---|---|---|---|---|
| `wazuh-server` | SIEM (Manager + Indexer + Dashboard, all-in-one) | Ubuntu Server 24.04 LTS | 8 GB (upgraded from 4 GB) | 4 (upgraded from 2) | 50 GB dynamic |
| `endpoint-linux-wazuh` | Monitored endpoint (Wazuh Agent) | Ubuntu Server 24.04 LTS | 2 GB | 2 | 20 GB dynamic |
| `kali` | Attacker (pre-existing lab VM) | Kali Linux | — | — | — |

### Software installed
- Wazuh 4.14.6 (Manager, Indexer, Dashboard, Agent)
- OpenSSH Server (on both Linux VMs, for remote administration via Windows Terminal)
- Hydra (pre-installed on Kali)
- `sshpass` (on the endpoint, for a manual scripted brute-force variant)

---

## Fase 1 — Infrastructure Setup

**Goal:** Stand up an isolated VirtualBox lab with a SIEM server that can both reach the internet and be reached by other lab VMs and the host.

**Approach:**
The Manager VM (`wazuh-server`) was created first. Before deciding on networking, we reasoned through the difference between VirtualBox's **NAT**, **Internal Network**, and **Host-only Network** modes:

| Mode | VM↔VM | VM↔Host | VM↔Internet |
|---|---|---|---|
| NAT | ❌ (each VM isolated) | ⚠️ limited | ✅ |
| Internal Network | ✅ | ❌ | ❌ |
| Host-only Network | ✅ | ✅ | ❌ |

Since the lab needed internet access (for package installs), inter-VM communication (agent→manager), **and** host→dashboard access, a **dual-adapter** setup (NAT + Host-only) was chosen over a single-mode network.

```bash
mkdir wazuh-siem-lab
cd wazuh-siem-lab
mkdir docs/setup docs/ingest docs/alerts endpoints/linux-agent endpoints/windows-agent notes
```

VM creation used VirtualBox's *Unattended Installation* feature (Ubuntu Server 24.04, 8 GB RAM / 4 vCPU / 50 GB dynamically-allocated disk — RAM/CPU was later increased from the initial 4 GB/2 vCPU baseline after a resource-related install failure described in Phase 2).

![00 buat struktur folder](docs/setup/00-buat-struktur-folder.png)
![01 vm name and os selection](docs/setup/01-vm-name-and-os-selection.png)
![02 unattended install config](docs/setup/02-unattended-install-config.png)
![03 vm hardware allocation](docs/setup/03-vm-hardware-allocation.png)
![04 vm hard disk config](docs/setup/04-vm-hard-disk-config.png)
![05 network adapter1 nat](docs/setup/05-network-adapter1-nat.png)
![06 network adapter2 hostonly](docs/setup/06-network-adapter2-hostonly.png)
![07 grub boot menu](docs/setup/07-grub-boot-menu.png) *(optional)*
![08 first successful login](docs/setup/08-first-successful-login.png)

After first boot, the dual-adapter design was validated:

```bash
ip a
```

Two interfaces confirmed: `enp0s3` (NAT, `10.0.2.15`) and `enp0s8` (Host-only, `192.168.56.104`).

![09 cek ip address vm](docs/setup/09-cek-ip-address-vm.png)

**Troubleshooting note:** during setup, an accidental **hard power-off** of the VM (instead of a graceful shutdown) later caused Wazuh's internal database service to fail on the next boot — see Phase 6 for the full recovery process. This reinforced the importance of graceful shutdowns for stateful services.

For convenient remote administration (and to enable normal clipboard copy/paste, which does not work over the raw VirtualBox console on a headless Ubuntu Server), OpenSSH Server was installed and enabled on both Linux VMs, allowing all subsequent work to be done from Windows Terminal instead of the VirtualBox GUI console:

```bash
sudo apt update && sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

![17 ssh server wazuh server](docs/setup/17-ssh-server-wazuh-server.png)
![18 ssh login dari windows](docs/setup/18-ssh-login-dari-windows.png)

**Result:** A working Ubuntu Server VM with verified dual-network connectivity and remote SSH access, ready for Wazuh installation.

---

## Fase 2 — Wazuh Installation (Manager + Indexer + Dashboard)

**Goal:** Install the full Wazuh all-in-one stack on `wazuh-server`.

**Approach:**
Wazuh's official all-in-one installer script was downloaded and verified against the official domain (HTTPS + official `packages.wazuh.com` source) before execution — an early attempt to also verify a `.sha512` checksum failed because Wazuh does not actually publish a separate checksum file for the installer script; the official documentation instead relies on HTTPS transport security alone.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

**Troubleshooting — Indexer startup timeout:**
The first install attempt failed:

```
[ 2984.465432] watchdog: BUG: soft lockup - CPU#0 stuck for 295s!
Job for wazuh-indexer.service failed because a timeout was exceeded.
```

![10 instalasi wazuh gagal timeout](docs/setup/10-instalasi-wazuh-gagal-timeout.png)

Root cause analysis: Wazuh Indexer is OpenSearch/Java-based and requires substantial CPU/RAM during its first JVM startup. The original allocation (4 GB RAM / 2 vCPU) was insufficient in a virtualized environment. Resources were increased to **8 GB RAM / 4 vCPU** (the host had 32 GB physical RAM available, so this was safe).

![11 troubleshoot ram 8gb](docs/setup/11-troubleshoot-ram-8gb.png)
![12 troubleshoot cpu 4core](docs/setup/12-troubleshoot-cpu-4core.png)

Re-running the installer succeeded on the second attempt:

```
25/07/2026 01:34:23 INFO: wazuh-indexer service started.
25/07/2026 01:41:53 INFO: wazuh-manager service started.
25/07/2026 01:42:44 INFO: wazuh-dashboard service started.
25/07/2026 01:47:44 INFO: Installation finished.
```

![13 instalasi wazuh berhasil](docs/setup/13-instalasi-wazuh-berhasil.png)
![14 cek ip setelah instalasi](docs/setup/14-cek-ip-setelah-instalasi.png)

The dashboard was accessed over HTTPS from the Windows host via the Host-only IP (accepting the self-signed certificate warning, expected since Wazuh generates its own SSL cert):

```
https://192.168.56.104
```

![15 login page wazuh dashboard](docs/setup/15-login-page-wazuh-dashboard.png)
![16 wazuh dashboard overview](docs/setup/16-wazuh-dashboard-overview.png)

**Result:** A fully running Wazuh all-in-one instance, accessible from the host browser, already generating baseline self-monitoring alerts (Wazuh Manager monitors its own host by default via an internal agent).

---

## Fase 3 — Endpoint Deployment & Log Forwarding

**Goal:** Deploy a second VM as a monitored endpoint and connect it to the Manager via the Wazuh Agent.

**Approach:**
A second VM (`endpoint-linux-wazuh`, 2 GB RAM / 2 vCPU / 20 GB) was built from a fresh Ubuntu Server ISO (rather than cloning `wazuh-server`, since a clone would have carried over the full Manager/Indexer/Dashboard stack — unnecessarily heavy for a lightweight monitored endpoint) using identical dual-adapter networking so it would land on the same `192.168.56.0/24` subnet.

![00 vm name and os selection](endpoints/linux-agent/00-vm-name-and-os-selection.png)
![01 unattended install config](endpoints/linux-agent/01-unattended-install-config.png)
![02 vm hardware allocation](endpoints/linux-agent/02-vm-hardware-allocation.png)
![03 vm hard disk config](endpoints/linux-agent/03-vm-hard-disk-config.png)
![04 network adapter1 nat](endpoints/linux-agent/04-network-adapter1-nat.png)
![05 network adapter2 hostonly](endpoints/linux-agent/05-network-adapter2-hostonly.png)
![06 first successful login](endpoints/linux-agent/06-first-successful-login.png)
![07 cek ip address vm](endpoints/linux-agent/07-cek-ip-address-vm.png)

Connectivity between the two VMs (`192.168.56.106` ↔ `192.168.56.104`) was validated with `ping` before installing the agent:

![08 ping test ke wazuh server](endpoints/linux-agent/08-ping-test-ke-wazuh-server.png)

**Troubleshooting — boot order:** after a first reboot, the VM briefly failed to boot from disk and attempted PXE network boot instead (`PXE-E06: Option ROM requires DDIM support`). This was resolved by confirming Hard Disk was the top boot priority in VirtualBox → System → Motherboard.

![09 fix boot order](endpoints/linux-agent/09-fix-boot-order.png)

OpenSSH was enabled here as well, for the same remote-administration reasons as Phase 1:

![10 ssh server setup](endpoints/linux-agent/10-ssh-server-setup.png)
![11 ssh login dari windows](endpoints/linux-agent/11-ssh-login-dari-windows.png)

The Wazuh Agent repository and package were installed manually (unlike the Manager, the Agent does not come with a repo pre-configured):

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee -a /etc/apt/sources.list.d/wazuh.list

sudo apt update
```

![12 setup repository wazuh](endpoints/linux-agent/12-setup-repository-wazuh.png)

```bash
WAZUH_MANAGER="192.168.56.104" sudo apt-get install wazuh-agent -y
```

![13 instalasi wazuh agent berhasil](endpoints/linux-agent/13-instalasi-wazuh-agent-berhasil.png)

**Troubleshooting — environment variable dropped by `sudo`:** the agent installed but failed to start:

```
ERROR: (4112): Invalid server address
ERROR: (1215): No client configured.
```

Root cause: `sudo` resets the calling shell's environment by default, so `WAZUH_MANAGER=...` placed **before** `sudo` never reached the privileged process. The correct order would have been `sudo WAZUH_MANAGER="..." apt-get install ...`. As an equally valid fix, the Manager address was instead set directly in the agent's config file:

![14 error agent gagal start](endpoints/linux-agent/14-error-agent-gagal-start.png)

```bash
sudo nano /var/ossec/etc/ossec.conf
# <address>192.168.56.104</address>
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

![15 edit ossec conf](endpoints/linux-agent/15-edit-ossec-conf.png)
![16 config address manager](endpoints/linux-agent/16-config-address-manager.png)
![17 agent berhasil running](endpoints/linux-agent/17-agent-berhasil-running.png)

**Result:** the agent registered successfully and appeared as **Active** in the Wazuh Dashboard.

![18 agent terdaftar di dashboard](endpoints/linux-agent/18-agent-terdaftar-di-dashboard.png)
![19 overview agent active](endpoints/linux-agent/19-overview-agent-active.png)

---

## Fase 4 — Ingestion Verification

**Goal:** Confirm that logs from the new endpoint are actually being ingested and are searchable, not just that the agent shows "Active."

**Approach:**
The per-agent detail page (`Endpoints → endpoint-linux-wazuh`) was reviewed, showing system inventory, event-count evolution, MITRE ATT&CK mapping, vulnerability detection, and a Security Configuration Assessment (CIS Ubuntu 24.04 Benchmark) that ran automatically on agent registration.

![20 detail agent dashboard](endpoints/linux-agent/20-detail-agent-dashboard.png)

The **Threat Hunting** module (Dashboard + Events tabs) was used to confirm raw, searchable log entries existed for the new agent (not just aggregate counters), including CIS benchmark findings and standard authentication/session events.

![21 threat hunting dashboard](endpoints/linux-agent/21-threat-hunting-dashboard.png)
![22 threat hunting events log](endpoints/linux-agent/22-threat-hunting-events-log.png)

**Result:** confirmed the full ingestion pipeline (Agent → Manager → Indexer → Dashboard) was functioning end-to-end for the new endpoint.

---

## Fase 5 — Attack Simulation / Test Log Generation

**Goal:** Generate realistic SSH brute-force traffic against the lab to produce meaningful data for tuning, using two different methods for comparison.

**Approach — Method A: Hydra (from Kali)**
A small, deliberately trimmed username/password wordlist was built to keep the attack fast and within reasonable data-usage limits (the lab was run on a mobile data connection, not Wi-Fi):

```bash
cat << EOF > sshbf.txt
wazuh
root
admin
user
EOF

head -n 10 /usr/share/wordlists/rockyou.txt > passwordssshbf.txt
```

![00 kali network check](attack-simulation/ssh-bruteforce/00-kali-network-check.png)
![01 struktur folder](attack-simulation/ssh-bruteforce/01-struktur-folder.png)
![02 buat file username](attack-simulation/ssh-bruteforce/02-buat-file-username.png)
![03 cek wordlist password](attack-simulation/ssh-bruteforce/03-cek-wordlist-password.png)
![04 generate passwords list](attack-simulation/ssh-bruteforce/04-generate-passwords-list.png)

An initial run with the full 1,000-line password list and `-t 4` threads was estimated by Hydra at **~52 hours to complete** (SSH's per-attempt cryptographic handshake makes brute-forcing inherently slow, unlike lighter protocols). The wordlist was trimmed to 10 lines and thread count raised to `-t 16`, reducing runtime to under 30 seconds:

```bash
hydra -L sshbf.txt -P passwordssshbf.txt -t 16 192.168.56.104 ssh
```

```
[DATA] max 16 tasks per 1 server, overall 16 tasks, 40 login tries (l:4/p:10)
1 of 1 target completed, 0 valid password found
```

![05 eksekusi hydra selesai](attack-simulation/ssh-bruteforce/05-eksekusi-hydra-selesai.png)

**Approach — Method B: manual scripted loop (from `endpoint-linux-wazuh`)**
A second, sequential (non-parallel) attack was scripted using `sshpass` to compare detection behavior against Hydra's parallel approach:

```bash
sudo apt install -y sshpass
```

```bash
#!/bin/bash
PASSWORDS=("123456" "password" "admin123" "qwerty" "letmein" "wazuh123" "toor" "changeme" "wazuh1" "test123")
for pw in "${PASSWORDS[@]}"; do
    echo "Trying password: $pw"
    sshpass -p "$pw" ssh -o StrictHostKeyChecking=no -o ConnectTimeout=3 \
      wazuh@192.168.56.104 "echo connected" 2>&1 | grep -v "Warning"
done
```

**Result — detection comparison:** the two attack styles triggered **different rule sets** in Wazuh:

| Attack method | Pattern | Rules triggered |
|---|---|---|
| Hydra (parallel, 16 threads) | Burst of simultaneous attempts | `2502` – *syslog: User missed the password more than one time* (level 10) |
| Manual script (sequential) | One-at-a-time with small delays | `5763` – *sshd: brute force trying to get access to the system* (level 10), `5551` – *PAM: Multiple failed logins in a small period of time* (level 10) |

Both methods also generated large volumes of the low-severity **rule 5710** (*sshd: Attempt to login using a non-existent user*, level 5) — this became the central "noise" problem addressed in Phase 6.

![06 wazuh detect bruteforce attack](attack-simulation/ssh-bruteforce/06-wazuh-detect-bruteforce-attack.png)
![07 event log detail bruteforce](attack-simulation/ssh-bruteforce/07-event-log-detail-bruteforce.png)
![08 ssh loop manual berhasil](attack-simulation/ssh-bruteforce/08-ssh-loop-manual-berhasil.png)
![09 deteksi manual bruteforce rule berbeda](attack-simulation/ssh-bruteforce/09-deteksi-manual-bruteforce-rule-berbeda.png)

Wazuh's MITRE ATT&CK mapping automatically classified the traffic under **T1110 (Brute Force)** / **Password Guessing / SSH**, confirming the SIEM's built-in threat-intelligence enrichment.

**Result:** ~1,984 authentication-failure events were generated across both methods, providing a realistic, high-volume dataset for the tuning phase.

---

## Fase 6 — Alert & Rule Tuning

**Goal:** Reduce alert noise from rule `5710` and produce a faster, more sensitive custom detection for repeated non-existent-user login attempts — the central objective of this project.

**Approach:**
Analysis of the generated logs showed that rule **5710** (level 5) fired on *every single* failed attempt — thousands of times — while Wazuh's own escalation logic only kicked in after a much larger threshold. This is a textbook case of **alert fatigue**: high-volume, low-severity noise that can bury genuinely actionable high-severity alerts and desensitize a SOC analyst reviewing the feed daily.

Custom rules in Wazuh live in `/var/ossec/etc/rules/local_rules.xml` rather than the default ruleset (`/var/ossec/ruleset/rules/`), so that upstream Wazuh updates never overwrite locally authored logic, and custom rules can be trivially removed to revert to stock behavior. Custom rule IDs must be **≥ 100000** to avoid colliding with Wazuh's reserved built-in rule ID space.

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

First iteration:

```xml
<rule id="100002" level="10" frequency="8" timeframe="120">
  <if_matched_sid>5710</if_matched_sid>
  <description>SSH brute force attempt: multiple non-existent user login failures detected.</description>
  <group>authentication_failures,pci_dss_11.4,</group>
</rule>
```

![00 cek local rules default](docs/alerts/00-cek-local-rules-default.png)
![01 custom rule 100002 final](docs/alerts/01-custom-rule-100002-final.png)

Validated (without needing a full service restart) using Wazuh's built-in rule tester:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

![02 validasi rule logtest](docs/alerts/02-validasi-rule-logtest.png)

**Troubleshooting — Wazuh Manager not running (from a prior ungraceful shutdown):** `wazuh-logtest` failed with `Wazuh-logtest error when connecting with wazuh-analysisd`. Investigation revealed the Manager had crashed after the VM was hard-powered-off in an earlier session, leaving `wazuh-db` and other core services down while `wazuh-authd`/`wazuh-apid` remained stuck in a partially-running state:

```bash
sudo systemctl status wazuh-manager     # Active: failed (Result: timeout)
sudo tail -n 50 /var/ossec/logs/ossec.log
# wazuh-authd: ERROR: Unable to connect to socket 'queue/db/wdb'
sudo /var/ossec/bin/wazuh-control status
# wazuh-db not running...
```

![03 error logtest dan status awal](docs/alerts/03-error-logtest-dan-status-awal.png)
![04 cek ossec log error](docs/alerts/04-cek-ossec-log-error.png)
![05 cek status komponen wazuh](docs/alerts/05-cek-status-komponen-wazuh.png)
![06 filter log wazuh db](docs/alerts/06-filter-log-wazuh-db.png)

The fix was a full, explicit stop → verify → start cycle (a plain `systemctl restart` alone was later found to be insufficient — see below):

```bash
sudo /var/ossec/bin/wazuh-control stop
sudo /var/ossec/bin/wazuh-control status   # confirm everything "not running"
sudo /var/ossec/bin/wazuh-control start
sudo /var/ossec/bin/wazuh-control status   # confirm core services "is running"
```

![07 stop semua service wazuh](docs/alerts/07-stop-semua-service-wazuh.png)
![08 verifikasi semua service mati](docs/alerts/08-verifikasi-semua-service-mati.png)
![09 start ulang service wazuh](docs/alerts/09-start-ulang-service-wazuh.png)
![10 verifikasi service berhasil running](docs/alerts/10-verifikasi-service-berhasil-running.png)

**Iterative rule testing:** feeding the same sample SSH log into `wazuh-logtest` repeatedly (simulating repeated attempts) revealed an important discovery — **rule 100002 never fired**. Instead, Wazuh's own **built-in rule 5712** (also frequency-based, threshold of 8) fired first and "consumed" the correlated event window before the custom rule could trigger:

![11 test logtest 1x percobaan rule 5710](docs/alerts/11-test-logtest-1x-percobaan-rule-5710.png)
![12 logtest rule bawaan 5712 menang duluan](docs/alerts/12-logtest-rule-bawaan-5712-menang-duluan.png)

This was an important tuning insight: a custom rule with an identical threshold to an existing built-in rule is redundant. To make the custom rule meaningfully **faster** than Wazuh's stock detection, the frequency was lowered from 8 to **5** — sensitive enough to beat the default, but still high enough to avoid flagging normal human typos:

```xml
<rule id="100002" level="10" frequency="5" timeframe="120">
  <if_matched_sid>5710</if_matched_sid>
  <description>SSH brute force attempt; multiple non-existent user login failures detected.</description>
  <group>authentication_failures,pci_dss_11.4,</group>
</rule>
```

![13 revisi frequency jadi 5](docs/alerts/13-revisi-frequency-jadi-5.png)

Re-tested — rule `100002` fired correctly on the 5th repeated attempt this time:

![14 custom rule berhasil trigger](docs/alerts/14-custom-rule-berhasil-trigger.png)

The Manager was fully restarted to load the change into production (again using the full `wazuh-control stop`/`start` cycle rather than `systemctl restart`, after confirming the latter can silently skip reloading `wazuh-analysisd`):

![15 restart wazuh manager berhasil](docs/alerts/15-restart-wazuh-manager-berhasil.png)
![18 full stop wazuh manager](docs/alerts/18-full-stop-wazuh-manager.png)
![19 full start wazuh manager berhasil](docs/alerts/19-full-start-wazuh-manager-berhasil.png)

**Live validation:** both attack methods (Hydra and the manual script) were re-run against `wazuh-server` and the results checked against a live Dashboard query:

```
rule.id: 100002
```

![16 validasi serangan manual setelah tuning](docs/alerts/16-validasi-serangan-manual-setelah-tuning.png)
![17 validasi serangan hydra setelah tuning](docs/alerts/17-validasi-serangan-hydra-setelah-tuning.png)
![20 validasi final serangan manual](docs/alerts/20-validasi-final-serangan-manual.png)
![21 validasi final serangan hydra](docs/alerts/21-validasi-final-serangan-hydra.png)
![22 validasi final custom rule berhasil di dashboard](docs/alerts/22-validasi-final-custom-rule-berhasil-di-dashboard.png)

**Result:** custom rule `100002` fired **6 times** in the Dashboard, all at level 10. The math checked out: of the 50 total login attempts across both attack methods, only 30 used a genuinely non-existent username (the rest used the valid `wazuh` username with a wrong password, which is handled by a different rule path) — `30 ÷ 5 (frequency) = 6` alerts, exactly matching the observed count. This confirmed the custom rule was firing correctly, not randomly.

**Tuning outcome:** the SIEM now escalates a non-existent-user SSH brute-force pattern to a high-severity alert after **5 failed attempts within 2 minutes**, instead of the stock **8-attempt** threshold — a **37.5% faster** detection time, directly satisfying the project's original goal of *"cutting the noise and surfacing real threats faster."*

---

## Fase 7 — Active Response (Automated Containment)

**Goal:** Automatically block an attacking IP address when the tuned custom rule (`100002`) fires, and safely reverse that block (both automatically and manually).

**Approach:**
Active Response commands execute on the **Agent side** (the victim host), not on the Manager — the Manager only decides *when* to trigger a response and pushes the command over the existing Manager↔Agent channel. For this reason, the attack target was switched from `wazuh-server` itself to `endpoint-linux-wazuh`, to avoid the Manager ever blocking access to itself and to better mirror a realistic scenario (a Manager is rarely internet/SSH-exposed in production; monitored endpoints are).

Before enabling blocking, a whitelist of hosts that must never be blocked was identified:
- `192.168.56.104` (`wazuh-server`, the Manager itself)
- `192.168.56.106` (`endpoint-linux-wazuh`, self)
- `192.168.56.1` (the Windows host's Host-only adapter IP, used for all remote SSH administration)

![00 cek ip windows host](docs/active-response/00-cek-ip-windows-host.png)

Wazuh ships a set of pre-registered Active Response commands (`firewall-drop`, `disable-account`, `host-deny`, `route-null`, etc.) — conceptually similar to pre-defined functions that only need to be "called" by name, rather than written from scratch. `firewall-drop` was selected, which manages `iptables` rules automatically.

```xml
<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>defined-agent</location>
  <agent_id>001</agent_id>
  <rules_id>100002</rules_id>
  <timeout>600</timeout>
</active-response>
```

`location: defined-agent` (targeting a specific agent ID) was chosen over `location: all` for this stage of the lab — with only one endpoint currently deployed, the practical effect is identical, but `defined-agent` gives tighter, more auditable control while the detection logic is still being validated. `location: all` was noted as a stronger fit for a lab with multiple monitored endpoints (documented under Next Steps).

![01 tambah config active response](docs/active-response/01-tambah-config-active-response.png)

The Manager was restarted using the same full stop/start cycle established in Phase 6:

![02 full stop wazuh manager](docs/active-response/02-full-stop-wazuh-manager.png)
![03 full start wazuh manager berhasil](docs/active-response/03-full-start-wazuh-manager-berhasil.png)

**Validation — automatic block:**

```bash
hydra -L sshbf.txt -P passwordssshbf.txt -t 16 192.168.56.106 ssh
```

The Dashboard confirmed the full detection chain firing in sequence: `5710` (noise) → `100002` (tuned correlation) → **`651` — "Host Blocked by firewall-drop Active Response."**

![04 dashboard active response berhasil](docs/active-response/04-dashboard-active-response-berhasil.png)

A subsequent SSH attempt from Kali hung indefinitely with no error message — expected behavior for an `iptables DROP` rule (as opposed to `REJECT`), which silently discards packets rather than notifying the sender, denying the attacker even the information that they have been blocked:

![05 kali terblokir ssh hang](docs/active-response/05-kali-terblokir-ssh-hang.png)

Confirmed directly on the endpoint's firewall table:

```bash
sudo iptables -L INPUT -n --line-numbers
```

```
num  target     prot opt source               destination
1    DROP       0    --  192.168.56.102       0.0.0.0/0
```

![06 iptables rule block verified](docs/active-response/06-iptables-rule-block-verified.png)

**Validation — automatic unblock (timeout):** the block was confirmed present shortly after the attack, then re-checked several minutes later and found to have been automatically removed by Wazuh once the configured `600`-second timeout elapsed — with no manual intervention:

![07 verifikasi auto unblock timeout](docs/active-response/07-verifikasi-auto-unblock-timeout.png)

**Validation — manual unblock:** a fresh attack was triggered to re-populate the block, and the rule was then removed manually before its timeout expired, to confirm an administrator can always intervene early if needed:

```bash
hydra -L sshbf.txt -P passwordssshbf.txt -t 16 192.168.56.106 ssh
sudo iptables -L INPUT -n --line-numbers   # confirm DROP rule present
sudo iptables -D INPUT 1                   # delete rule #1
sudo iptables -L INPUT -n --line-numbers   # confirm empty again
```

![08 serangan ulang untuk test unblock](docs/active-response/08-serangan-ulang-untuk-test-unblock.png)
![09 unblock manual berhasil](docs/active-response/09-unblock-manual-berhasil.png)

A final SSH attempt from Kali confirmed access was restored (host key verification prompt appeared, followed by a normal password prompt — no hang):

![10 verifikasi akses normal setelah unblock](docs/active-response/10-verifikasi-akses-normal-setelah-unblock.png)

**Result:** a fully closed-loop, automated blue-team pipeline was demonstrated end-to-end: detection → correlation → tuned escalation → automated containment → automatic and manual reversal — without any manual intervention required during the active-block phase.

---

## Final Verification / Proof of Completion

The complete kill-chain was validated live against the Wazuh Dashboard, in this exact sequence, across multiple independent test runs:

1. Kali/endpoint generates SSH login failures against `endpoint-linux-wazuh`
2. Rule `5710` (built-in, level 5) fires per individual failed attempt
3. Custom rule `100002` (level 10, `frequency=5`, `timeframe=120s`) escalates after the 5th failure — faster than Wazuh's stock 8-attempt threshold (rule `5712`)
4. Active Response `firewall-drop` automatically adds an `iptables DROP` rule for the attacker's IP on the agent
5. Attacker's SSH connections silently hang (packets dropped, no rejection notice)
6. Block is automatically removed after 600 seconds, **or** manually removed via `iptables -D`
7. Access is confirmed restored after unblock

![06 iptables rule block verified](docs/active-response/06-iptables-rule-block-verified.png)
![10 verifikasi akses normal setelah unblock](docs/active-response/10-verifikasi-akses-normal-setelah-unblock.png)

---

## Lessons Learned

- **Alert volume ≠ alert value.** A rule firing thousands of times a day is not automatically "working correctly" — if it can't be reasonably reviewed by a human, it becomes noise that hides genuine threats. Tuning is about *proportional* severity and correlation, not just detection.
- **Custom rules should be checked against built-in rules first.** Wazuh already ships correlation logic for common patterns; a custom rule with an identical threshold does nothing new. The value of tuning came from deliberately setting a *tighter* threshold than the default, not from reinventing detection from scratch.
- **`if_sid` vs. `if_matched_sid` matter.** The former checks a single event in isolation; the latter checks accumulated frequency across a time window — understanding this distinction was essential to making `frequency`/`timeframe` correlation work at all.
- **`systemctl restart` is not always a true restart.** During recovery from a crashed Manager, `systemctl restart wazuh-manager` reported services as "already running" without reloading `wazuh-analysisd`, meaning rule changes were silently *not* applied. A full `wazuh-control stop` → verify → `start` cycle was required to guarantee the ruleset was reloaded.
- **Ungraceful VM shutdowns have real consequences.** A single forced power-off corrupted the internal `wazuh-db` state enough to break the Manager on next boot — a reminder that stateful services (databases, message queues) need clean shutdown paths even in a disposable lab environment.
- **Attack *pattern* affects which detection fires.** Parallel (Hydra) vs. sequential (scripted) brute-force attempts against the same target triggered different built-in Wazuh rules (`2502` vs. `5763`/`5551`), a useful reminder that detection engineering has to account for varied attacker tooling, not just attack *type*.
- **`DROP` vs `REJECT` in Active Response.** Wazuh's `firewall-drop` silently discards attacker traffic rather than rejecting it — denying the attacker feedback about whether the block succeeded, which is a deliberate and realistic blue-team design choice.
- **Environment variables and `sudo` don't always compose the way you expect.** `VAR=value sudo command` and `sudo VAR=value command` behave differently because `sudo` resets the environment by default — a small but easy-to-miss gotcha that caused a real agent misconfiguration during the project.

---

## Next Steps

The following were identified as valuable extensions but deliberately scoped out to keep this iteration focused:

1. **Windows Agent deployment** — install and register a Wazuh Agent on a Windows endpoint to demonstrate cross-platform monitoring (folder `endpoints/windows-agent/` was reserved in the project structure but not yet used).
2. **Active suppression of rule `5710` noise** — beyond adding a correlation rule on top, explore Wazuh's options for actually reducing/suppressing the volume of low-severity events reaching the dashboard (e.g. `<options>no_log</options>`), while preserving them for forensic retention.
3. **Investigate the "valid user, wrong password" detection path** — 20 of the 50 simulated login attempts used the real `wazuh` username with an incorrect password; the specific rule capturing these was not conclusively isolated via Dashboard queries and remains an open investigation.
4. **Author a fully custom Active Response script** (rather than using Wazuh's built-in `firewall-drop`) for a bespoke containment action.
5. **Compare `location: all` vs. `location: defined-agent`** for Active Response once a second monitored endpoint exists in the lab, to evaluate the trade-offs of blast radius vs. coverage discussed during design.
6. **Before/after vulnerability comparison** — the endpoint reported 35 Critical / 606 High vulnerabilities (mostly from an un-upgraded kernel/packages at initial deployment); running `apt upgrade` and re-scanning would make for a compelling before/after tuning narrative.

---

## Notes on Sanitization

All IP addresses in this document (`192.168.56.0/24`, `10.0.2.x`) belong to a private, isolated VirtualBox Host-only/NAT lab network and are not sensitive. The Wazuh Dashboard admin password generated during installation (`uT11KjOJmSS1K.MrBVKo+XGhKc3ywQMl`) was displayed once during the session and should be treated as `<REDACTED_DASHBOARD_PASSWORD>` if this document is published — **rotate this password before sharing any related screenshots publicly.** No other credentials, API keys, or personally identifying information were found in the source material for this writeup.
