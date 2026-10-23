## 🗂️ Project Snapshot — Wazuh SIEM Home Lab

> Deskripsi project lengkap — siap dipakai sebagai context di LinkedIn, CV, resume, atau web portofolio.

**Nama Project:** Wazuh SIEM Home Lab (SIEM Log Analysis & Alert Tuning)
**Periode:** Juli 2026 — masih berjalan (fase inti selesai, lanjut ke fase upgrade)
**Kategori:** Blue Team / SOC / Detection Engineering
**Stack/Tools:** Wazuh 4.14.6 (Manager, Indexer, Dashboard, Agent), VirtualBox, Ubuntu Server 24.04 LTS, Kali Linux, Hydra, sshpass, iptables, rockyou.txt

**Deskripsi:**
Home lab yang mensimulasikan workflow SOC Analyst/Detection Engineer end-to-end: membangun infrastruktur SIEM terisolasi, menghubungkan endpoint yang dipantau, men-generate serangan SSH brute force nyata dari mesin attacker terpisah, menganalisis alert yang dihasilkan, menulis custom detection rule untuk mempercepat deteksi, lalu mengotomasi respons containment (block IP) lengkap dengan mekanisme auto/manual unblock. Project ini juga mencakup dokumentasi profesional bergaya blue-team (Incident Report, Playbook, Detection Rule) sebagai bahan portofolio.

**Yang sudah dikerjakan:**
- Membangun 3 VM terisolasi (Manager, endpoint Linux, attacker Kali) dengan jaringan dual-adapter (NAT + Host-only)
- Instalasi penuh Wazuh Manager, Indexer, Dashboard, dan Agent, termasuk troubleshooting resource dan crash service
- Dua skenario simulasi serangan SSH brute force (Hydra paralel dan script manual sekuensial)
- Penulisan custom correlation rule yang mendeteksi brute force 37.5% lebih cepat dari default Wazuh
- Implementasi Active Response otomatis (block IP via iptables) dengan validasi auto-unblock dan manual unblock
- Dokumentasi lengkap: README, 5 writeup per fase, 1 Incident Report blue-team style lengkap dengan Playbook dan Detection Rule, catatan belajar, dan roadmap upgrade

**Angka & bukti konkret:**
- 7 fase project inti selesai dan tervalidasi end-to-end
- 1 custom detection rule (ID 100002) diautorkan dan diuji
- 2 metode simulasi serangan, total 50 percobaan login per siklus uji penuh
- ~1.984 event authentication-failure dianalisis pada tahap awal tuning
- 6 alert custom rule tervalidasi dari 30 percobaan yang memenuhi kriteria (matematis sesuai: 30 ÷ 5 = 6)
- Deteksi brute force 37.5% lebih cepat dari threshold default Wazuh (5 percobaan vs 8 percobaan)
- ~97 file screenshot dokumentasi tersebar di 6 folder bukti kerja
- 9 file markdown dokumentasi (1 README, 6 writeup, 2 notes)

**5 Screenshot Terpenting:**
*(5 screenshot yang paling merepresentasikan hasil project — paling layak ditampilkan di portofolio)*

| No | Nama File (dengan path folder) | Apa yang ditunjukkan |
|---|---|---|
| 1 | `assets/active-response/04-dashboard-active-response-berhasil.png` | Rantai deteksi penuh di Dashboard: noise rule → custom rule tuning → Active Response block, dalam satu alur |
| 2 | `assets/alerts/14-custom-rule-berhasil-trigger.png` | Custom detection rule berhasil ter-trigger setelah proses debugging dan revisi threshold |
| 3 | `assets/active-response/06-iptables-rule-block-verified.png` | Bukti teknis IP penyerang berhasil diblokir otomatis lewat iptables |
| 4 | `assets/setup/16-wazuh-dashboard-overview.png` | Tampilan awal Wazuh Dashboard setelah instalasi all-in-one berhasil |
| 5 | `assets/linux-agent/18-agent-terdaftar-di-dashboard.png` | Endpoint berhasil terdaftar sebagai agent aktif, menandai pipeline log forwarding berfungsi |

---

## 📋 Project Progress Checklist — Wazuh SIEM Home Lab

**Terakhir diupdate:** 2026-10-23

### Infrastructure & Environment Setup

| Status | Item | Catatan |
|---|---|---|
| ✅ | Setup 3 VM VirtualBox (Manager, endpoint Linux, Kali) | Dual-adapter NAT + Host-only |
| ✅ | Instalasi Ubuntu Server 24.04 (unattended install) | |
| ✅ | Troubleshooting PXE boot order issue | |
| ✅ | Setup SSH remote administration dari Windows Terminal | Untuk Manager dan endpoint |

### Wazuh Installation

| Status | Item | Catatan |
|---|---|---|
| ✅ | Instalasi Wazuh all-in-one (Manager + Indexer + Dashboard) | |
| ✅ | Troubleshooting resource shortage (RAM/CPU) saat Indexer gagal start | 4GB/2vCPU → 8GB/4vCPU |
| ✅ | Verifikasi akses Dashboard dari browser host | |

### Endpoint Deployment & Log Forwarding

| Status | Item | Catatan |
|---|---|---|
| ✅ | Deploy VM endpoint kedua (Linux) | |
| ✅ | Instalasi Wazuh Agent + setup repository manual | |
| ✅ | Troubleshooting `WAZUH_MANAGER` env variable ke-reset oleh `sudo` | |
| ✅ | Verifikasi agent status Active di Dashboard | |

### Ingestion Verification

| Status | Item | Catatan |
|---|---|---|
| ✅ | Eksplorasi Threat Hunting (Dashboard + Events) | |
| ✅ | Verifikasi SCA, Vulnerability Detection, MITRE ATT&CK mapping | |

### Attack Simulation

| Status | Item | Catatan |
|---|---|---|
| ✅ | Brute force SSH via Hydra (parallel attack) | |
| ✅ | Brute force SSH via script manual `sshpass` (sequential) | |
| ✅ | Perbandingan rule yang ter-trigger antar metode serangan | |

### Alert & Rule Tuning

| Status | Item | Catatan |
|---|---|---|
| ✅ | Analisis alert fatigue pada rule bawaan 5710 | |
| ✅ | Penulisan custom rule di `local_rules.xml` (ID 100002) | |
| ✅ | Debug `if_sid` vs `if_matched_sid` | |
| ✅ | Revisi frequency 8→5 setelah discovery rule bawaan 5712 "menang duluan" | |
| ✅ | Validasi pakai `wazuh-logtest` | |
| ✅ | Troubleshooting Wazuh Manager crash (`wazuh-db` corrupt) | |
| ✅ | Validasi custom rule di Dashboard dengan data serangan nyata | |

### Active Response

| Status | Item | Catatan |
|---|---|---|
| ✅ | Setup Active Response `firewall-drop` terikat ke custom rule | |
| ✅ | Setup whitelist IP penting | |
| ✅ | Validasi auto-block (iptables DROP) | |
| ✅ | Validasi auto-unblock setelah timeout | |
| ✅ | Validasi manual unblock | |
| ✅ | Validasi akses kembali normal setelah unblock | |

### Documentation

| Status | Item | Catatan |
|---|---|---|
| ✅ | README.md + 5 file writeup per fase | |
| ✅ | Incident Report blue-team style lengkap | IOC, Timeline, Root Cause, Remediation |
| ✅ | Section Playbook & Detection Rule | |
| ✅ | Catatan belajar/rangkuman materi | `notes/learning-summary.md` |
| ✅ | Checklist & roadmap upgrade | `notes/upgrade-roadmap.md` |
| 🔄 | Reorganisasi struktur repo (`writeups/`, `assets/`, `notes/`) | File sudah disiapkan, belum dikonfirmasi dieksekusi di repo lokal |

### Upgrade Roadmap — Belum Dimulai

| Status | Item | Catatan |
|---|---|---|
| ⬜ | Integrasi Wazuh Agent di endpoint Windows | Endpoint sudah tersedia |
| ⬜ | Simulasi RDP brute force di Windows | |
| ⬜ | Custom rule untuk Windows Event ID (misal 4625) | |
| ⬜ | Active Response untuk Windows | |
| ⬜ | Uji File Integrity Monitoring secara langsung | |
| ⬜ | Hardening SSH (key-only auth, fail2ban) | |
| ⬜ | Simulasi Nmap port/service scanning | |
| ⬜ | Simulasi web application attack (SQLi/XSS) | Butuh VM/web app tambahan |
| ⬜ | Simulasi privilege escalation | |
| ⬜ | Simulasi malware/IOC (EICAR test file) | |
| ⬜ | Simulasi lateral movement Linux → Windows | |
| ⬜ | Timeline reconstruction forensik dari log | |
| ⬜ | Chain of custody practice | |
| ⬜ | Export log untuk analisis offline | |
| ⬜ | Custom decoder (`local_decoder.xml`) | |
| ⬜ | Rule exclusion/override tambahan | |
| ⬜ | Custom dashboard visualization | |
| ⬜ | Perbandingan `location: all` vs `defined-agent` | |
| ⬜ | Investigasi rule untuk "username valid, password salah" | |
| ⬜ | Before/after vulnerability scan comparison | |

---

### 📊 Summary

| Metric | Jumlah |
|---|---|
| ✅ Selesai | 33 item |
| 🔄 Sedang berjalan | 1 item |
| ⬜ Belum dimulai | 20 item |
| **Total** | **54 item** |
| **Progress** | **61%** |

**Terakhir selesai di:** Dokumentasi lengkap (README, 5 writeup, Incident Report, catatan belajar, roadmap upgrade) sudah digenerate; struktur repo final (`writeups/`, `assets/`, `notes/`) sudah dirancang dan file-nya disiapkan, tinggal dieksekusi migrasinya di repo lokal.
**Selanjutnya:** Eksekusi migrasi struktur folder di repo lokal, lalu mulai item upgrade prioritas — integrasi Wazuh Agent di endpoint Windows.
