# TEK1314 Keamanan Siber — Kelompok 01 Kelas B

Simulasi keamanan siber IoT pada infrastruktur replika dan data dummy. Pengujian hanya berjalan pada VM lab terisolasi; bukan sistem produksi.

## Identitas dan Peran

- Mata kuliah: TEK1314 — Keamanan Siber
- Program studi: Teknologi Rekayasa Komputer
- Project Lead: Satria Febrian (J0404241011)
- Red Team: Indra Fata Azzaky (J0404241017)
- Blue Team: Aldy Alfaridzi Damas (J0404231150), Athaya Rifqi Andriansyah (J0404241012)

## Scope Lab

- Internal: `192.168.1.0/24`
- Management: `192.168.56.0/24`
- Target: `target-server` — `192.168.1.5` / `192.168.56.10`
- Red Team VM: `cyber-workstation` — `192.168.1.100`
- IoT device: `iot-node` — `192.168.1.201`
- Practical/archive documents: [`docs/praktikum/`](docs/praktikum/)
- SIEM/NIDS: `sec-onion` — sniffer/SPAN, tanpa IP pada jaringan simulasi
- Domain: `dreamsheep.local`

## Navigasi Artefak

| Area | Artefak | Status |
|---|---|---|
| Design | [`docs/design/`](docs/design/) | Tersedia |
| Blue Team | [`docs/blueteam/`](docs/blueteam/) | Tersedia |
| Red Team | [`docs/redteam/`](docs/redteam/) | Tersedia |
| Purple Team | [`docs/purple-team-log.md`](docs/purple-team-log.md) | Template |
| Incident report | [`docs/final/incident_report_draft.md`](docs/final/incident_report_draft.md) | Draft; export PDF manual |
| Final presentation | [`docs/presentation-final/slides_outline.md`](docs/presentation-final/slides_outline.md) | Outline; export PDF manual |
| Video demo | `[Link Video Demo YouTube/Google Drive Unlisted]` | Belum diisi |

## Checklist Pengumpulan

- [x] Struktur `design`, `blueteam`, `redteam`
- [x] Blueprint, IP plan, topologi, hardening, attack plan tertaut
- [x] Template Purple Team Loop tahap 1–5
- [x] Draft incident report NIST SP 800-61r2
- [x] Outline 10–12 slide
- [ ] Isi bukti runtime dan hasil retest
- [ ] Ekspor `incident_report.pdf`
- [ ] Ekspor `slides.pdf`
- [ ] Isi tautan video demo unlisted
- [ ] Review akses repository untuk dosen

## Prinsip Operasional

Tidak ada brute force, flood, persistence, penghapusan log/data, stealth, atau pengujian jaringan eksternal. Finding harus memiliki bukti runtime, batas trust yang jelas, dampak terukur, dan rekomendasi yang dapat diverifikasi.
