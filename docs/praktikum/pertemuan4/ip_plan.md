# IP Address Plan — Kelompok 01 (Kelas B)

**Mata Kuliah:** TEK1314 - Keamanan Siber
**Fase:** Design Phase (Pertemuan 4)
**Subnet Kelompok:** `192.168.1.0/24`
**Netmask:** `255.255.255.0`
**Broadcast:** `192.168.1.255`
**Gateway Segmen:** `192.168.1.1`

## Tabel Alokasi IP

| Hostname                  | IP Address      | OS Direncanakan | Peran                          |
|----------------------------|-----------------|------------------|---------------------------------|
| Target Server (Korban)     | 192.168.1.5     | Metasploitable2  | Aset yang diamankan (Blue Team) |
| Attacker Node              | 192.168.1.100   | Kali Linux       | Red Team                        |
| Monitoring Node            | 192.168.1.200   | Security Onion   | Blue Team — NSM / IDS           |
| Gateway / Switch (opsional)| 192.168.1.1     | —                | Router/L2 switch segmen         |

## Catatan Skenario (Blue Team)

- **Skenario serangan:** Web Application (fokus utama port 80/443), dengan layanan tambahan pada
  Metasploitable2 yang tetap perlu dipantau: FTP (21), SSH (22), dan MySQL (3306).
- **Penempatan Monitoring Node:** Security Onion dipasang pada port *mirror/SPAN* di switch segmen
  kelompok, sehingga dapat mengawasi seluruh trafik antara Attacker Node dan Target Node tanpa
  menjadi bagian dari jalur komunikasi utama (out-of-band monitoring).
- **Alasan pemilihan Metasploitable2:** ringan secara resource (cocok untuk keterbatasan RAM laptop),
  dan sudah memiliki banyak layanan rentan bawaan (web app, FTP backdoor, MySQL) sehingga Red Team
  punya beberapa opsi vektor serangan untuk didemonstrasikan.
- Tabel routing sederhana belum diperlukan karena seluruh node berada pada satu segmen/subnet yang sama
  (`192.168.1.0/24`); tidak ada routing antar-VLAN pada fase ini.

> Catatan: OS target dan skenario di atas adalah rekomendasi awal Blue Team. Konfirmasikan kembali
> dengan Red Team dan Lead sebelum tahap implementasi agar keputusan tercatat sesuai arahan panduan
> (poin II.3 dan Tugas Lead).
