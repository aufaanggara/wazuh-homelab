# SIEM Log Analysis & Alert Tuning — Wazuh Home Lab

**Type:** Personal Home Lab Project
**Category:** Blue Team / SOC / Detection Engineering
**Stack:** Wazuh 4.14.6 (Manager, Indexer, Dashboard, Agent) · VirtualBox · Ubuntu Server 24.04 · Kali Linux

A hands-on home lab simulating the core workflow of a SOC Analyst / Detection Engineer: deploy a SIEM, forward logs from a monitored endpoint, generate realistic attack traffic, analyze the resulting alerts, tune detection rules to cut noise, and automate a defensive response.

Inspired by a TryHackMe "Project 01" prompt — *"Deploy Splunk, ELK, or Wazuh to collect and correlate logs across your environment, then tune alert thresholds to cut the noise and surface real threats faster."*

---

## Lab Topology

<img src="docs/setup/06-network-adapter2-hostonly.png" width="700">

| VM | Role | OS | RAM | vCPU | IP |
|---|---|---|---|---|---|
| `wazuh-server` | SIEM (Manager + Indexer + Dashboard) | Ubuntu Server 24.04 | 8 GB | 4 | `192.168.56.104` |
| `endpoint-linux-wazuh` | Monitored endpoint (Wazuh Agent) | Ubuntu Server 24.04 | 2 GB | 2 | `192.168.56.106` |
| `kali` | Attacker (lab-internal) | Kali Linux | — | — | `192.168.56.102` |
| Windows Host | Administration | Windows 11 | — | — | `192.168.56.1` |

All VMs run dual VirtualBox adapters — **NAT** (internet access) + **Host-only** (`192.168.56.0/24`, isolated VM↔VM and VM↔Host communication).

---

## Project Phases

| # | Phase | Summary | Docs |
|---|---|---|---|
| 1–2 | Infrastructure & Wazuh Installation | Built the Manager VM, networking, and installed the full Wazuh all-in-one stack | [`docs/01-setup.md`](docs/01-setup.md) |
| 3–4 | Endpoint Deployment & Ingestion Verification | Deployed a second VM, registered it as a Wazuh Agent, verified log ingestion end-to-end | [`docs/02-endpoint.md`](docs/02-endpoint.md) |
| 5 | Attack Simulation | Generated real SSH brute-force traffic (Hydra + manual script) from Kali | [`docs/03-attack-simulation.md`](docs/03-attack-simulation.md) |
| 6 | Alert & Rule Tuning | Authored a custom correlation rule that detects brute force 37.5% faster than Wazuh's default | [`docs/04-alert-tuning.md`](docs/04-alert-tuning.md) |
| 7 | Active Response | Automated firewall blocking of attacking IPs, with auto/manual unblock validated | [`docs/05-active-response.md`](docs/05-active-response.md) |

**Related document:** [`incident-report-ssh-bruteforce.md`](incident-report-ssh-bruteforce.md) — a standalone blue-team style Incident Report + Playbook + Detection Rule write-up of the SSH brute-force detection/response exercise, formatted for portfolio review.

---

## Key Result

A custom Wazuh correlation rule (`100002`) was authored to detect repeated SSH login attempts using non-existent usernames, escalating to a high-severity alert after **5 failed attempts in 120 seconds** — faster than Wazuh's built-in default threshold of **8 attempts** (rule `5712`). This tuned alert was then bound to an **Active Response** action that automatically blocks the attacking IP via `iptables`, with the block automatically lifted after a 10-minute cooldown (and a validated manual-override path). The full loop — detect → correlate → tune → block → auto-unblock — was validated end-to-end against two independent brute-force methods.

<img src="docs/active-response/04-dashboard-active-response-berhasil.png" width="700">

---

## Lessons Learned

- Alert volume is not a proxy for detection quality — tuning is about *proportional* severity, not just raising more alerts.
- A custom rule must be checked against existing built-in rules; the real tuning win came from a *tighter* threshold than Wazuh's default, not from writing a rule from scratch.
- `systemctl restart wazuh-manager` is not always a true restart — a full `wazuh-control stop` → `start` cycle was required to guarantee rule changes were reloaded.
- `DROP` vs. `REJECT` in Active Response is a deliberate design choice — silently dropping attacker traffic denies them feedback on whether the block succeeded.

---

## Future Work

1. Deploy a Windows Agent to demonstrate cross-platform monitoring
2. Explore active suppression of low-severity noise (beyond adding a correlation layer on top)
3. Isolate the detection path for "valid username, wrong password" attempts (currently unconfirmed)
4. Author a fully custom Active Response script
5. Compare `location: all` vs. `defined-agent` once a second endpoint exists
6. Run a before/after vulnerability scan comparison post-`apt upgrade`
