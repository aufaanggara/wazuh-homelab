# Wazuh SIEM Home Lab

Home lab deploying and tuning a Wazuh SIEM — from infrastructure setup to automated attack detection and response.

---

## Deskripsi

Project ini mensimulasikan workflow SOC Analyst/Detection Engineer end-to-end: membangun infrastruktur SIEM terisolasi, menghubungkan endpoint yang dipantau, men-generate serangan SSH brute force nyata, menganalisis alert, menulis custom detection rule untuk mempercepat deteksi, dan mengotomasi respons containment. Diinspirasi dari prompt project TryHackMe "SIEM Log Analysis & Alert Tuning", dan dikembangkan lebih jauh dengan tahap Active Response.

## Tech Stack & Tools

- **Wazuh 4.14.6** — SIEM stack (Manager, Indexer, Dashboard, Agent)
- **VirtualBox** — virtualisasi seluruh VM lab
- **Ubuntu Server 24.04 LTS** — OS Manager dan endpoint
- **Kali Linux** — mesin attacker simulasi
- **Hydra** — tool brute force paralel
- **sshpass** — scripting brute force manual sekuensial
- **iptables** — mekanisme blocking Active Response

## Struktur Folder

```
wazuh-siem-lab/
├── README.md
│
├── writeups/
│   ├── 01-setup.md
│   ├── 02-endpoint.md
│   ├── 03-attack-simulation.md
│   ├── 04-alert-tuning.md
│   ├── 05-active-response.md
│   └── incident-report-ssh-bruteforce.md
│
├── assets/
│   ├── setup/
│   ├── linux-agent/
│   ├── windows-agent/
│   ├── ssh-bruteforce/
│   ├── alerts/
│   └── active-response/
│
└── notes/
    ├── troubleshooting.md
    ├── learning-summary.md
    └── upgrade-roadmap.md
```

## Progress

| Fase | Status |
|---|---|
| Infrastructure & Environment Setup | ✅ Selesai |
| Wazuh Installation | ✅ Selesai |
| Endpoint Deployment & Log Forwarding | ✅ Selesai |
| Ingestion Verification | ✅ Selesai |
| Attack Simulation | ✅ Selesai |
| Alert & Rule Tuning | ✅ Selesai |
| Active Response | ✅ Selesai |
| Documentation | ✅ Selesai |
| Repo Restructuring | 🔄 Berjalan |
| Windows Endpoint Integration & Upgrades | ⬜ Belum |

**Progress keseluruhan: 61%**

## Dokumentasi

Writeup lengkap per fase tersedia di folder [`writeups/`](writeups/):

1. [Infrastructure Setup & Wazuh Installation](writeups/01-setup.md)
2. [Endpoint Deployment & Ingestion Verification](writeups/02-endpoint.md)
3. [Attack Simulation](writeups/03-attack-simulation.md)
4. [Alert & Rule Tuning](writeups/04-alert-tuning.md)
5. [Active Response](writeups/05-active-response.md)
6. [Incident Report — SSH Brute Force](writeups/incident-report-ssh-bruteforce.md) — dokumentasi blue-team style lengkap dengan IOC, timeline, root cause, remediation, playbook, dan detection rule

Catatan tambahan (bukan untuk portofolio, referensi pribadi) ada di folder [`notes/`](notes/).

## Disclaimer

Project ini dibuat untuk tujuan pembelajaran dan portofolio pribadi. Seluruh simulasi serangan (brute force, dsb.) dijalankan sepenuhnya di dalam environment lab terisolasi (jaringan VirtualBox Host-only), menargetkan sistem milik sendiri, dan tidak diarahkan ke sistem atau jaringan pihak lain mana pun.
