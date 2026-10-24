[← Kembali ke README](README.md) · [Lanjut: Deteksi Awal →](02-investigasi-deteksi-awal.md)

# Laporan Insiden — SSH Brute Force

**Tanggal:** Juli 2026
**Platform:** Home lab pribadi (Wazuh SIEM)
**Tingkat Kesulitan:** Menengah
**Kategori:** Intrusion Attempt — SSH Brute Force
**Tools yang Digunakan:** Wazuh 4.14.6, Hydra, sshpass, iptables, Kali Linux

## Ringkasan Eksekutif

Antara 27-30 Juli 2026, endpoint `endpoint-linux-wazuh` (`192.168.56.106`) kena serangan brute force SSH dari host `192.168.56.102` (Kali Linux yang jadi simulasi attacker di lab ini). Totalnya 50 percobaan login per kampanye serangan, pakai kombinasi username valid dan tidak valid. Semua percobaan gagal, nggak ada satupun kredensial yang kebobol. Wazuh berhasil deteksi aktivitas ini, dan setelah dilakukan tuning, custom rule yang dibuat bisa eskalasi alert 37,5% lebih cepat dibanding deteksi bawaan Wazuh. Respons otomatis (block IP) langsung jalan begitu alert level tinggi muncul, dan blokirnya kebuka otomatis setelah 10 menit. Nggak ada data yang bocor, nggak ada lateral movement — ini murni latihan validasi kemampuan deteksi dan respons lab sendiri.

## Indikator Kompromi (IOC)

| Tipe IOC | Nilai | Keterangan |
|---|---|---|
| IP Penyerang | `192.168.56.102` | VM Kali Linux, dipakai sebagai simulasi attacker |
| IP Target (endpoint) | `192.168.56.106` | `endpoint-linux-wazuh`, target utama buat validasi tuning dan Active Response |
| IP Target (awal) | `192.168.56.104` | `wazuh-server`, sempat jadi target di tahap validasi awal sebelum dipindah ke endpoint |
| Protokol/Port | SSH / TCP 22 | Vektor serangan |
| Tool penyerang | Hydra v9.6 | Brute force paralel, pakai `-t 16` thread |
| Tool penyerang | Script bash + sshpass | Brute force sekuensial, 10 percobaan |
| Username dicoba | `wazuh`, `root`, `admin`, `user` | 1 dari 4 valid di target, sisanya nggak ada |
| Password dicoba | Potongan 10 baris dari `rockyou.txt` + 10 password umum | Nggak ada yang cocok |
| Rule Wazuh (noise) | `5710` — sshd: Attempt to login using a non-existent user | Level 5, nyala tiap 1 percobaan gagal (~1.984 event) |
| Rule Wazuh (bawaan, threshold 8) | `5712` | Nggak dipakai lagi setelah custom rule dibuat lebih sensitif |
| Rule Wazuh (pola paralel) | `2502` — syslog: User missed the password more than one time | Level 10, nyala pas pakai Hydra |
| Rule Wazuh (pola sekuensial) | `5763` / `5551` | Level 10, nyala pas pakai script manual |
| Custom rule | `100002` — SSH brute force attempt; multiple non-existent user login failures detected | Level 10, `frequency=5`, `timeframe=120s` — dibuat sendiri selama investigasi |
| Active Response | `651` — Host Blocked by firewall-drop Active Response | Level 3, block otomatis via iptables |

## Alur Kejadian (Timeline)

| Waktu (UTC) | Kejadian |
|---|---|
| 2026-07-27 ~08:00-08:30 | Kampanye brute force pertama dijalankan ke `wazuh-server` (Hydra 40 percobaan + script manual 10 percobaan), buat ngumpulin data awal tuning |
| 2026-07-27 08:29:16-08:29:28 | Custom rule `100002` (setelah tuning) nyala 6 kali dari dataset ini, konfirmasi logic korelasinya jalan |
| 2026-07-28 03:21 | Wazuh Manager di-restart penuh buat load konfigurasi Active Response final |
| 2026-07-30 07:37:36 | Target dipindah ke `endpoint-linux-wazuh`. Rule `5710` dan `5760` nyala berulang, rule `100002` nyala di 07:37:36.139 |
| 2026-07-30 07:37:36.507-540 | Active Response `651` nyala dua kali, konfirmasi containment otomatis langsung jalan setelah alert tuning muncul |
| 2026-07-30 ~20:37 | Attacker nyoba konek SSH lagi ke `192.168.56.106` dan sesinya hang total, konfirmasi `iptables DROP` aktif |
| 2026-07-30 (+10 menit dari block) | Rule `iptables` buat `192.168.56.102` kecatat otomatis hilang setelah timeout 600 detik, nggak ada intervensi manual |
| 2026-07-30 ~20:57 | Kampanye kedua dijalankan ulang buat trigger block lagi, kali ini dibuka manual (`iptables -D INPUT 1`) sebelum timeout |
| 2026-07-30 (langsung setelah unblock manual) | Attacker konfirmasi akses SSH ke `192.168.56.106` balik normal |

---

[← Kembali ke README](README.md) · [Lanjut: Deteksi Awal →](02-investigasi-deteksi-awal.md)
