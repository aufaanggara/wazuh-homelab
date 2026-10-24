[← README](README.md) · [← Sebelumnya: Dampak & Remediasi](05-analisis-dampak-dan-remediasi.md)

# Playbook & Detection Rule

## Playbook: SSH Brute Force Alert (Rule 100002)

**Pemicu:** Alert Wazuh dengan `rule.id: 100002` (SSH brute force attempt; multiple non-existent user login failures detected), level 10.

**Langkah Triase:**
1. Cek source IP dari alert (`data.srcip`) dan agent yang kena (`agent.name`).
2. Liat di Threat Hunting → Events, apakah IP yang sama juga trigger rule korelasi lain (`2502`, `5763`, `5551`) — ini nunjukin seberapa intens pola serangannya (paralel atau sekuensial).
3. Cek apakah Active Response (`651`) udah jalan otomatis. Kalau belum, cek lagi binding `<active-response>` di `ossec.conf` buat agent terkait, kemungkinan ada yang kedisable atau salah konfigurasi.
4. Cocokin source IP dengan whitelist yang udah ditentuin (`192.168.56.104`, `192.168.56.106`, `192.168.56.1` di lab ini) — biar nggak salah nganggep traffic legitimate sebagai false positive.
5. Review volume rule `5710` dari pasangan source-target yang sama di beberapa menit sebelumnya, buat mastiin ini emang pola berkelanjutan, bukan cuma satu kejadian aneh doang.

**Kriteria Eskalasi:**
- Ada login berhasil (`sshd: authentication success`) dari IP yang sama setelah burst brute force — ini tanda kemungkinan kompromi beneran dan harus langsung dieskalasi, nggak peduli status Active Response.
- Source IP nggak dikenal/nggak match ke host manapun yang ada di jaringan.
- Active Response gagal jalan meskipun rule `100002` udah nyala — artinya jalur containment-nya rusak dan perlu block manual.

**Tindakan Penahanan:**
- Otomatis: Active Response `firewall-drop` block source IP lewat `iptables DROP` selama 600 detik (default).
- Manual (kalau perlu buka lebih cepat atau tahan lebih lama):
```bash
sudo iptables -L INPUT -n --line-numbers   # cek nomor rule-nya
sudo iptables -D INPUT <nomor_rule>        # hapus lebih awal
# atau, kalau mau permanen sampai direview manual:
sudo iptables -I INPUT -s <ip_attacker> -j DROP
```

**Indikator False Positive:**
- Source IP masuk whitelist (Manager, endpoint sendiri, atau host admin) — kemungkinan besar salah ketik kredensial berulang dari script yang salah konfigurasi, bukan serangan beneran.
- Volume rendah (kurang dari 5 kegagalan dalam jendela waktu) — nggak akan trigger `100002` sesuai desain rule-nya, tapi tetap layak direview kalau digabung sama indikator mencurigakan lain.
- Percobaan cuma pakai username valid tunggal (nggak ada username yang nggak ada) — pola ini nggak kecover sama rule `100002` (yang basisnya `5710`) dan perlu ditriase terpisah.

## Detection Rule

**Platform:** Wazuh (custom ruleset, `local_rules.xml`)
**Tujuan:** Mendeteksi percobaan brute force SSH yang berkelanjutan pakai username yang nggak ada, lebih cepat dari threshold korelasi bawaan Wazuh.

**Rule/Query:**
```xml
<group name="local,syslog,sshd,">
  <rule id="100002" level="10" frequency="5" timeframe="120">
    <if_matched_sid>5710</if_matched_sid>
    <description>SSH brute force attempt; multiple non-existent user login failures detected.</description>
    <group>authentication_failures,pci_dss_11.4,</group>
  </rule>
</group>
```

**Penjelasan Logic:**
Rule ini nggak parsing log mentah langsung — dia nempel di atas rule bawaan `5710` yang udah lebih dulu match pola log "login pakai user yang nggak ada" (level 5, nyala tiap kejadian). Pakai `<if_matched_sid>5710</if_matched_sid>` (korelasi berbasis frekuensi, beda dari `<if_sid>` yang cuma cek event tunggal), Wazuh ngelacak berapa kali rule `5710` udah nyala baru-baru ini. Begitu udah 5 kali dalam jendela 120 detik (`frequency=5`, `timeframe=120`), rule `100002` ikut nyala di level 10 — cukup tinggi buat dianggap actionable, dan lewat binding Active Response juga langsung trigger containment IP otomatis. Threshold-nya sengaja diset di bawah rule bawaan `5712` (threshold 8) biar rule custom ini yang "menang" duluan dalam korelasi, soalnya rule pertama yang konsumsi jendela event tertentu bakal nyuppress rule korelasi lain buat event yang sama.

**Catatan False Positive:**
- User yang beneran lupa username-nya terus nyoba beberapa kemungkinan lama dalam waktu singkat bisa aja ikut trigger rule ini meskipun bukan serangan. Di environment yang turnover usernya tinggi, threshold ini mungkin perlu dinaikin dikit atau digabung sama pengecekan terhadap daftar username yang emang valid.
- Rule ini cuma nyangkut jalur **username nggak valid** (`5710`), jadi nggak nangkep brute force yang pakai username valid tapi password salah berulang — itu kecover rule bawaan lain (`5760`/`5503` di investigasi ini) dan belum ada custom rule korelasi khusus buat jalur itu.

---

[← README](README.md) · [← Sebelumnya: Dampak & Remediasi](05-analisis-dampak-dan-remediasi.md)
