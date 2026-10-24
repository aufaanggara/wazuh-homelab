[← README](README.md) · [← Sebelumnya: Active Response](04-investigasi-active-response.md) · [Lanjut: Playbook & Detection Rule →](06-playbook-dan-detection-rule.md)

# Analisis Dampak & Remediasi

## Analisis Akar Masalah

**Penyebab utama:** endpoint bisa diakses SSH tanpa rate-limiting atau lockout policy apapun di level OS. SSH emang nggak rate-limit sendiri secara default, jadi semua beban deteksi dan respons harus ditanggung di level SIEM.

**Faktor pendukung:** threshold korelasi bawaan Wazuh (`5712`, butuh 8 percobaan) ternyata cukup lambat buat use case ini — rule default-nya belum dioptimasi buat kasus spesifik kayak brute force username-tidak-valid yang pengen dideteksi lebih cepat.

## Penilaian Dampak

- **Sistem terdampak:** cuma `endpoint-linux-wazuh`, satu endpoint di lab terisolasi.
- **Data yang mungkin terekspos:** nggak ada. Semua 50+ percobaan login gagal total (`0 valid password found`).
- **Jangkauan lateral movement:** nggak ada. Ini lab tertutup, nggak ada host lain yang bisa dijangkau selain yang udah di-whitelist.

## Remediasi & Rekomendasi

**Tindakan segera (udah diimplementasi selama investigasi ini):**
- Custom rule `100002` buat eskalasi lebih cepat dibanding threshold default
- Active Response `firewall-drop` buat block otomatis IP penyerang
- Whitelist IP penting biar containment nggak nembak diri sendiri

**Jangka pendek:**
- Investigasi jalur deteksi buat kasus "username valid tapi password salah" — 20 dari 50 percobaan pakai username `wazuh` yang valid, dan belum ketemu query/rule spesifik yang nangkep pola ini secara jelas
- Terapin pola tuning yang sama ke endpoint lain kalau nanti ditambah
- Pertimbangin ganti `location: defined-agent` jadi `all` kalau lab udah punya lebih dari satu endpoint

**Jangka panjang:**
- Tambahin proteksi di level OS juga (fail2ban, atau disable password auth sepenuhnya, pakai SSH key doang) sebagai lapisan kedua di luar deteksi SIEM
- Formalin custom rule ini jadi playbook biar triase alert serupa konsisten ke depannya
- Rutin jalanin `apt upgrade` dan vulnerability scan di endpoint — ditemukan 35 Critical dan 606 High vulnerability pas investigasi, nggak langsung terkait insiden ini tapi tetap nambah attack surface yang perlu dibenerin

## Pelajaran yang Didapat

- Volume alert tinggi bukan jaminan deteksi bagus — rule `5710` yang nyala di tiap event individual nggak kasih sinyal actionable apapun sampai dikorelasiin pakai threshold waktu.
- Custom rule harus dicek dulu lawan rule bawaan yang udah ada buat pola serupa. Threshold yang sama persis itu sia-sia; yang bikin ada gunanya justru pas threshold-nya dibuat lebih ketat dari default.
- `if_sid` dan `if_matched_sid` itu beda total — yang pertama cek event tunggal, yang kedua cek akumulasi riwayat. Salah pakai salah satunya bikin rule korelasi nggak pernah nyala.
- `systemctl restart wazuh-manager` ternyata nggak selalu reload rule baru dengan benar — harus pakai siklus penuh `wazuh-control stop` → `start` biar rule-nya kepastian ke-load.
- Pola serangan (paralel vs sekuensial) bikin rule Wazuh yang beda yang nyala, meskipun sama-sama brute force SSH — jadi validasi deteksi emang harus nyoba beberapa gaya serangan, bukan cuma satu.
- `iptables DROP` yang dipakai Active Response Wazuh sengaja nggak kasih tau attacker kalau dia keblokir — desain defensif yang masuk akal buat ditiru kalau bikin automasi respons sendiri ke depannya.

---

[← README](README.md) · [← Sebelumnya: Active Response](04-investigasi-active-response.md) · [Lanjut: Playbook & Detection Rule →](06-playbook-dan-detection-rule.md)
