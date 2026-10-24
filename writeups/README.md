# Laporan Insiden — SSH Brute Force di Wazuh Home Lab

Ini laporan insiden hasil investigasi serangan SSH brute force yang disimulasikan sendiri di home lab Wazuh (bukan room/platform eksternal, tapi lab buatan sendiri dengan skenario serangan nyata). Ditulis dengan format incident report ala blue team, biar bisa dipakai juga sebagai portofolio.

**Tanggal:** Juli 2026
**Kategori:** Intrusion Attempt — SSH Brute Force
**Tools:** Wazuh 4.14.6, Hydra, sshpass, iptables, Kali Linux

## Ringkasan Singkat

Endpoint `endpoint-linux-wazuh` diserang brute force SSH dari dua metode berbeda (Hydra dan script manual). Semua percobaan gagal, tapi dari sini ketahuan masalah besar: rule default Wazuh terlalu berisik dan terlalu lambat buat deteksi serangan jenis ini. Jadi dibikin custom rule buat tuning deteksinya, terus ditambah Active Response biar IP penyerang otomatis diblokir. Semuanya udah divalidasi jalan beneran, bukan cuma teori.

## Daftar Isi

1. [Ringkasan Eksekutif & IOC](01-ringkasan-dan-ioc.md)
2. [Investigasi: Deteksi Awal & Masalah Noise](02-investigasi-deteksi-awal.md)
3. [Investigasi: Tuning Custom Rule](03-investigasi-tuning-rule.md)
4. [Investigasi: Containment Otomatis (Active Response)](04-investigasi-active-response.md)
5. [Analisis Dampak & Remediasi](05-analisis-dampak-dan-remediasi.md)
6. [Playbook & Detection Rule](06-playbook-dan-detection-rule.md)

## Highlight Hasil

Custom rule yang dibuat berhasil mendeteksi brute force **37,5% lebih cepat** dibanding rule bawaan Wazuh (5 percobaan vs 8 percobaan sampai alert level tinggi muncul). Setelah itu, IP penyerang otomatis diblokir pakai `iptables` lewat Active Response, dan blokirnya juga otomatis kebuka lagi setelah 10 menit — jadi bukan cuma deteksi doang, tapi sampai ke tahap respons otomatis.
