# Susunan Materi Presentasi PBL — Planning sampai Fase 2

**Proyek:** Simulasi Keamanan Siber IoT — Kelompok 01
**Tujuan presentasi:** Menjelaskan alur proyek dari perencanaan sampai rencana pengujian Fase 2, termasuk batas bahwa implementasi dan pengujian belum dilakukan.
**Saran durasi:** 10–15 menit, 16 slide.

---

## Slide 1 — Judul

**Judul:** Perancangan Simulasi Keamanan Siber IoT: Planning sampai Fase 2
**Subjudul:** Implementasi hardening, perencanaan validasi kerentanan, dan kesiapan pemulihan

**Isi:**
- TEK1314 — Keamanan Siber
- Kelompok 01 — Kelas B
- Nama anggota dan dosen pengampu
- Tanggal presentasi

**Visual:** Diagram empat fase sebagai garis waktu; tandai Fase 2 sebagai posisi saat ini.

**Narasi:** Presentasi menjelaskan progres dan rancangan proyek, bukan hasil eksploitasi atau hasil pengujian runtime.

---

## Slide 2 — Latar Belakang

**Isi:**
- Sistem IoT mengirim telemetri melalui MQTT ke server.
- Server menjalankan broker MQTT, API, reverse proxy Nginx, dan PostgreSQL dalam container.
- Sistem dipantau Security Onion melalui Zeek dan Suricata.
- Proyek menggabungkan pengamanan, pengujian keamanan terotorisasi, deteksi, pemulihan, dan patch.

**Visual:** Sensor → broker/API → database; jalur jaringan dipantau sensor SIEM.

**Narasi:** Fokus bukan hanya mencari celah, tetapi memahami bagaimana kontrol bekerja dan bagaimana tim memulihkan layanan.

---

## Slide 3 — Tujuan dan Ruang Lingkup

**Tujuan:**
1. Merancang topologi lab dan aliran telemetri.
2. Menentukan baseline hardening untuk host, jaringan, container, dan layanan.
3. Merencanakan evaluasi keamanan yang terotorisasi dan terukur.
4. Menyiapkan proses deteksi, pemulihan, patch, dan retest.

**Scope aset:**
- `target-server`: `192.168.1.5`, management `192.168.56.10`
- `iot-node`: `192.168.1.201`
- `cyber-workstation`: `192.168.1.100`
- `sec-onion`: tanpa IP (sniffer/SPAN)

**Batas:** Lab VM terisolasi; tidak mencakup host eksternal atau jaringan produksi.

**Visual:** Tabel aset dengan peran dan alamat.

---

## Slide 4 — Alur Fase Proyek

**Urutan:**
1. Planning PBL — definisi masalah, tujuan, aset, topologi, dan aturan.
2. Fase 1: Implementing & Hardening — membangun lab dan menerapkan baseline keamanan.
3. Fase 2: Zero Day Vulnerabilities — merencanakan evaluasi keamanan terotorisasi pada sistem yang telah dibangun.
4. Fase 3: Recovery and Security Patch — memulihkan layanan, memperbaiki temuan, menerapkan patch, dan retest.
5. Ending — menyusun hasil, evaluasi, demonstrasi, dan kesimpulan.

**Status saat ini:** Fase 2 masih tahap planning. Fase 1 belum diimplementasikan; karena itu belum ada hasil runtime atau temuan terkonfirmasi.

**Visual:** Timeline; tandai fase yang direncanakan dan posisi proyek saat ini.

**Narasi:** “Zero Day Vulnerabilities” adalah nama fase proyek. Status zero-day tidak boleh disimpulkan hanya dari nama fase atau konfigurasi yang tampak lemah.

---

## Slide 5 — Arsitektur Lab yang Direncanakan

**Segmen jaringan:**
- Internal `sim-cyber-net`: `192.168.1.0/24` untuk trafik IoT, target, dan pengujian.
- Host-Only `192.168.56.0/24` untuk management dan akses Security Onion.
- Target juga direncanakan memiliki interface NAT; akses dan alamat NAT berada di luar scope pengujian.

**Alur utama:**
- `iot-node` → MQTT `1883/TCP` → `target-server`
- Nginx `80/TCP` → API `3000/TCP` → PostgreSQL `5432/TCP` (antar-container)
- SPAN/mirror → `sec-onion` (Zeek/Suricata)

**Visual:** Diagram topologi sederhana dengan garis solid untuk komunikasi dan garis putus-putus untuk monitoring.

**Catatan:** Konfigurasi/topologi adalah rancangan dokumen; perlu divalidasi saat implementasi.

---

## Slide 6 — Komponen Sistem

| Komponen | Fungsi yang direncanakan |
|---|---|
| Ubuntu Server 22.04 | Host target dan Docker stack |
| Mosquitto | Broker MQTT untuk telemetri |
| Node.js API | Memproses telemetri dan menyediakan endpoint API |
| PostgreSQL | Penyimpanan data telemetri; tanpa port host pada Compose yang ditampilkan |
| Nginx | Reverse proxy HTTP ke API |
| Alpine Linux | Publisher telemetri melalui OpenRC |
| Security Onion | Pemantauan jaringan dan analisis event melalui Zeek/Suricata |
| Kali/LabVM | Workstation pengujian terotorisasi |

**Visual:** Diagram blok service di dalam batas host/container.

---

## Slide 7 — Fase 1: Rencana Implementasi & Hardening

**Domain hardening:**
- Identitas dan akses: nonaktifkan root SSH, batasi percobaan, least privilege.
- Jaringan: default-deny inbound; batasi SSH, HTTP, MQTT; batasi port database.
- Kernel: SYN cookies, reverse-path filtering, nonaktifkan redirect/source routing sesuai baseline.
- Container: minimalkan port publik, privilege, dan mount; PostgreSQL hanya pada Docker network.
- Deteksi: Fail2ban, Auditd, log host/container ke SIEM, sinkronisasi waktu.
- Patch: pembaruan keamanan dan `unattended-upgrades`.

**Status:** Belum diterapkan; semua butir adalah rencana konfigurasi.

**Visual:** Enam domain sebagai checklist berstatus “Direncanakan”.

---

## Slide 8 — Fase 1: Baseline dan Kriteria Siap Uji

**Sebelum Fase 2 dimulai:**
- VM dan jaringan lab berjalan sesuai topologi; tidak ada bridged adapter.
- Layanan dan port yang diharapkan terdokumentasi.
- Firewall dan interface binding diverifikasi.
- MQTT dan API memakai kontrol akses yang disepakati.
- PostgreSQL tidak terpapar ke host/jaringan tanpa kebutuhan.
- Security Onion menangkap trafik yang relevan; waktu dan log tersinkron.
- Snapshot dan prosedur restore sudah diuji.

**Visual:** Checklist go/no-go.

**Narasi:** Jika baseline atau visibilitas SIEM belum siap, pengujian keamanan ditunda agar hasil dapat dipercaya.

---

## Slide 9 — Fase 2: Tujuan dan Metodologi

**Tujuan:**
- Menguji apakah kontrol yang direncanakan bekerja pada sistem yang benar-benar telah dibangun.
- Mengidentifikasi konfigurasi berisiko dan kelemahan aplikasi secara terukur.
- Memastikan aktivitas uji terlihat pada log host dan Security Onion.

**Metodologi:**
1. Konfirmasi scope, izin, jendela, snapshot, dan stop condition.
2. Rekam baseline dan peta layanan.
3. Uji kontrol dengan langkah minimal dan terotorisasi.
4. Korelasikan bukti host, jaringan, dan SIEM.
5. Catat temuan, tingkat dampak, rekomendasi, dan kebutuhan verifikasi.

**Status:** Baru direncanakan; belum ada perintah yang dijalankan atau hasil yang diverifikasi.

---

## Slide 10 — Kandidat Vektor Uji Berdasarkan Blueprint

**Kandidat yang perlu diverifikasi setelah implementasi:**
- MQTT: blueprint menampilkan `allow_anonymous true` dan listener pada `0.0.0.0:1883`.
- API telemetri: route `/api/telemetry` tampak tidak memiliki autentikasi pada potongan kode blueprint.
- Transport: Nginx yang ditampilkan memakai HTTP port 80; TLS belum tampak pada konfigurasi contoh.
- Rahasia database: kredensial statis/default ditampilkan dalam contoh Compose dan connection string.
- Paparan jaringan: port Nginx/Mosquitto dipetakan tanpa alamat host spesifik; target memiliki beberapa interface.
- SSH: UFW membatasi SSH ke subnet management, sedangkan workstation penguji ada pada subnet internal.
- Logging: efektivitas log container, host, broker, dan forwarding ke SIEM perlu dibuktikan.

**Penting:** Ini hipotesis berbasis dokumen, bukan temuan runtime dan bukan otomatis zero-day.

**Visual:** Tabel vektor → kontrol terkait → bukti yang perlu dikumpulkan.

---

## Slide 11 — Makna “Zero-Day” dan Batas Klaim

**Definisi kerja:** Zero-day adalah kerentanan yang belum diketahui atau belum ditangani vendor/pemilik produk saat dieksploitasi, tanpa patch/perbaikan yang tersedia saat itu.

**Yang belum dapat diklaim:**
- Konfigurasi lemah dalam blueprint bukan otomatis zero-day.
- Belum ada verifikasi implementasi maupun eksploitasi terkontrol.
- Tidak ada CVE atau status vendor yang ditetapkan dari dokumen ini.

**Klaim yang tepat saat ini:** Fase 2 merencanakan validasi kandidat kelemahan; status dan dampaknya belum terkonfirmasi.

**Visual:** “Indikasi dokumen → verifikasi implementasi → bukti dampak → klasifikasi” dengan gerbang verifikasi.

---

## Slide 12 — Deteksi dan Bukti yang Direncanakan

| Aktivitas uji | Bukti yang dicari |
|---|---|
| Koneksi ke port yang disetujui | Zeek `conn.log`, alert Suricata jika rule cocok |
| Request HTTP/API | Zeek `http.log`, access/error log Nginx, log API |
| Event autentikasi SSH | `auth.log`/journal, event SIEM |
| Koneksi MQTT | Log Mosquitto dan metadata koneksi |
| Gangguan service (pada fase recovery) | Status container/service, gap heartbeat, alert downtime |

**Format bukti:** UTC timestamp, source/destination, test ID, perintah, hasil, event ID, log source, dan screenshot seperlunya.

**Visual:** Peta “aksi uji → log sumber → korelasi SIEM”.

---

## Slide 13 — Fase 3: Recovery dan Security Patch

**Rencana setelah validasi:**
1. Blue Team menerima temuan dan mengklasifikasikan dampak berdasarkan bukti.
2. Pulihkan service dari kondisi gagal menggunakan prosedur yang disetujui.
3. Terapkan perbaikan minimum: ACL/auth MQTT, auth API, pengelolaan secret, firewall/interface, TLS bila diperlukan, healthcheck dan alert.
4. Terapkan patch OS/dependency melalui proses maintenance yang disetujui.
5. Jalankan ulang test terkait; pastikan layanan sehat, kontrol bekerja, dan event terdeteksi.

**Metrik:** waktu deteksi, waktu pemulihan, keberhasilan retest, kelengkapan bukti.

**Visual:** Siklus deteksi → pemulihan → patch → retest.

---

## Slide 14 — Risiko, Batasan, dan Stop Condition

- Semua pengujian hanya di VM lab yang terisolasi dan diizinkan.
- Tidak menguji host eksternal, jaringan produksi, atau alamat NAT.
- Tidak melakukan stealth/evasion, brute force, flood, penghapusan log, persistence, atau akses data nyata.
- Hentikan jika ada host tak dikenal, trafik keluar scope, ketidakstabilan VM, risiko integritas data, atau SIEM overload.
- Perlu snapshot, console target, Lead, dan prosedur restore sebelum test berdampak pada ketersediaan.

**Visual:** Kotak scope dan stop-sign dengan trigger yang jelas.

---

## Slide 15 — Status Proyek dan Langkah Berikutnya

**Status saat presentasi:**
- Planning PBL: rancangan tujuan, scope, topologi, dan alur fase.
- Fase 1: rencana implementasi/hardening tersedia; belum diimplementasikan.
- Fase 2: rencana validasi dan dokumen test disusun; belum dilakukan pengujian runtime.
- Fase 3: rencana recovery/patch; belum dijalankan.

**Langkah berikutnya:**
1. Finalisasi topologi, scope, dan konfigurasi yang belum konsisten.
2. Implementasikan Fase 1 dan catat konfigurasi aktual.
3. Validasi baseline dan sensor SIEM.
4. Minta persetujuan sebelum pengujian Fase 2.
5. Gunakan bukti aktual untuk menetapkan temuan; jangan menyebut temuan zero-day tanpa dasar.

**Visual:** Kanban sederhana: Planning, Fase 1, Fase 2 (planning), Fase 3 (berikutnya).

---

## Slide 16 — Kesimpulan dan Tanya Jawab

**Kesimpulan:**
- Proyek berjalan bertahap: rencana → hardening → validasi → pemulihan/patch → evaluasi.
- Keamanan dinilai berdasarkan konfigurasi aktual dan bukti, bukan asumsi dokumen.
- Fase 2 saat ini masih perencanaan; implementasi dan verifikasi harus mendahului klaim kerentanan.

**Penutup:** Terima kasih — Tanya jawab.

**Visual:** Ringkasan lima fase dan pesan utama: “Uji terotorisasi, bukti terukur, pemulihan terverifikasi.”

---

## Catatan Penyusunan Slide

- Pertahankan status “belum diimplementasikan” secara konsisten; jangan tampilkan hipotesis sebagai hasil.
- Pisahkan “risiko dari blueprint” dari “finding terkonfirmasi”.
- Jangan menyebut kelemahan sebagai zero-day kecuali status unknown/unpatched telah diverifikasi.
- Gunakan diagram topologi yang konsisten dengan IP plan; tandai interface yang perlu konfirmasi.
- Cantumkan sumber internal pada catatan kaki slide terkait: `ip_plan.md`, `system_hardening_plan.md`, `blue_team_infrastructure_hardening_blueprint.md`, `attack_plan.md`, dan `attack_command.md`.
- Jika durasi dibatasi, gabungkan Slide 2–3, Slide 7–8, dan Slide 12–13; pertahankan slide status fase dan makna zero-day.

## Saran Pembagian Presenter

1. Presenter 1: Slide 1–5 — latar, tujuan, alur fase, topologi.
2. Presenter 2: Slide 6–8 — komponen dan rencana hardening.
3. Presenter 3: Slide 9–12 — metode Fase 2, kandidat vektor, zero-day, bukti.
4. Presenter 4: Slide 13–16 — recovery, batasan, status, kesimpulan.

Sesuaikan pembagian dengan jumlah anggota kelompok.
