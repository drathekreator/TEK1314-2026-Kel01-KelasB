# Attack Plan — Red-Team Discovery & Controlled Validation

**Proyek:** TEK1314 Keamanan Siber PBL, Kelompok 01
**Target:** `target-server` (`192.168.1.5`, `192.168.56.10`), `iot-node` (`192.168.1.201`), `sec-onion` (tanpa IP (sniffer/SPAN))
**Stack:** `/opt/iot-target-stack`, simulasi lokal `Dreamsheep-Simulation/`
**Attacker:** `cyber-workstation` / `192.168.1.100`
**Scope:** VirtualBox Internal Network `sim-cyber-net` dan Host-Only management. Tidak ada NAT, bridged, jaringan bersama, atau target eksternal.

## 1. Tujuan

1. Memetakan service yang benar-benar terbuka, bukan menganggap blueprint selalu diterapkan.
2. Memvalidasi kontrol perimeter: UFW, binding interface, TLS, SSH, MQTT, dan isolasi PostgreSQL.
3. Menguji batas aplikasi dengan request read-only dan payload benign.
4. Membuktikan dampak minimal: disclosure, unauthorized read/write, anonymous publish, atau konfigurasi yang tidak sesuai.
5. Menghasilkan bukti yang dapat dipakai Blue Team untuk patch dan retest.

## 2. Temuan Awal Berbasis Source

Status berikut **hipotesis sampai diuji runtime**.

| ID | Hipotesis | Source evidence | Dampak yang dicari | Prioritas |
|---|---|---|---|---|
| RT-01 | MQTT anonymous publish/read aktif | `mosquitto/config/mosquitto.conf:1-2` | Sensor spoof, telemetry poisoning, unauthorized topic read | High |
| RT-02 | MQTT bind `0.0.0.0` dan host publish `1883:1883` | `mosquitto.conf:1`, `docker-compose.yml:6-7` | Broker reachable dari subnet yang tidak semestinya | High |
| RT-03 | API `/api/readings` tanpa auth | `app/index.js:31` | Unauthorized telemetry disclosure | Medium |
| RT-04 | API menerima arbitrary JSON MQTT lalu menyimpan ke memory | `app/index.js:14-20` | Telemetry poisoning / resource-limited memory churn | Medium |
| RT-05 | Credentials PostgreSQL hardcoded | `docker-compose.yml:16-19,29` | Credential reuse jika source/config terbaca | Medium |
| RT-06 | Frontend login client-side dengan credential statis | `src/components/Login.jsx:4,11-19` | Login bypass by source inspection; no server-side session boundary | Medium |
| RT-07 | Firebase data read/write langsung dari browser | `src/hooks/useFirebaseSensor.js:51-73,154-160` | Unauthorized cloud data read/write jika Firebase rules lemah | High, needs runtime validation |
| RT-08 | Firebase config project exposed in client | `src/firebase.js:5-13` | Enables rule/config reconnaissance; not secret alone | Informational |
| RT-09 | HTTP redirects to HTTPS, but redirect leaks requested URL and TLS cert is self-signed | `nginx/default.conf:1-10` | Transport policy and certificate validation weakness | Low/Medium |
| RT-10 | Nginx accepts arbitrary `Host` via `_` server name | `nginx/default.conf:3,8` | Host-header routing confusion; validate only if app trusts Host | Low |
| RT-11 | API accepts any MQTT JSON shape and silently ignores malformed data | `app/index.js:15-20` | Integrity and monitoring blind spot | Low/Medium |
| RT-12 | `depends_on` lacks readiness checks; no Compose healthchecks | `docker-compose.yml:31-33,44-45` | Startup race and delayed recovery | Low |

**Catatan:** blueprint menyebut endpoint `/api/telemetry`, tetapi source simulasi menyediakan `/api/readings`. Uji kedua path, lalu catat perbedaan sebagai implementation drift.

## 3. Rules of Engagement

1. Snapshot semua VM sebelum test. Satu test aktif setiap waktu.
2. Discovery rate rendah. Gunakan timeout dan jumlah request terbatas.
3. Read-only dulu. Publish MQTT hanya ke topic lab `dreamsheep/telemetry` dan satu payload marker benign, setelah Lead menyetujui.
4. Tidak brute force, exploit framework, SQLi/XSS payload, flood, stress, persistence, credential spraying, data deletion, log tampering, atau exfiltrasi.
5. Jangan menguji Firebase write/toggle kecuali project dan data owner mengonfirmasi sandbox Firebase. Default: read-only validation.
6. Jangan menyentuh service stop tests AV-01—AV-07 kecuali prosedur gangguan lama sudah disetujui.
7. Stop jika ada data non-lab, trafik keluar subnet, service crash tidak pulih, atau alert/SIEM overload.
8. Finding confirmed hanya bila ada boundary, principal, resource, dan hasil yang teramati. Source-only tetap `needs validation`.

## 4. Alur Exercise

### Phase A — Preflight dan baseline

- `P-01`: verifikasi NIC, snapshot, route, source IP.
- `P-02`: fingerprint HTTP(S), endpoint, TLS, service port.
- `P-03`: cek status container, binding, UFW, SSH, MQTT log, Security Onion.

### Phase B — Low-impact discovery

- `RT-01/02`: MQTT anonymous access, topic scope, retained message behavior.
- `RT-03/04`: API route discovery, disclosure, method/content-type behavior, bounded malformed input.
- `RT-05`: configuration exposure only; tidak mengambil secret produksi.
- `RT-06`: review frontend bundle/source-visible auth behavior; no password spraying.
- `RT-07/08`: Firebase read rules dan metadata read-only dari browser/curl jika domain termasuk scope.
- `RT-09/10`: HTTP redirect, TLS versions/ciphers/certificate, Host behavior.
- `RT-11/12`: integrity and recovery observation; no resource exhaustion.

### Phase C — Minimal validation

1. Reproduce satu kali per hypothesis.
2. Capture request, response/status, timestamp UTC, source IP, target, and SIEM event.
3. Untuk MQTT poisoning, publish marker JSON sekali; record before/after and remove only marker if application stores it. Do not publish wildcard or production-like bulk data.
4. Untuk Firebase, stop at read-rule result. Write test needs explicit owner approval and disposable path.

### Phase D — Blue Team patch

Prioritas patch: disable anonymous MQTT + ACL/TLS; enforce API auth and schema validation; remove client-side trust; restrict Firebase rules; move DB credentials to secrets; bind ports to intended interface; add healthchecks/readiness; enforce certificate and Host policy; align docs with deployed routes.

### Phase E — Retest

Ulangi hanya test yang menghasilkan candidate. Expected: anonymous MQTT denied, unauthorized API denied, malformed payload rejected, Firebase unauthorized read/write denied, PostgreSQL absent externally, TLS policy passes, invalid Host rejected, and service recovery observable.

## 5. Acceptance Evidence

Untuk tiap ID simpan: run ID, operator, UTC start/end, VM snapshot, source IP, exact command, target, response, packet/log evidence, impact, cleanup, Blue Team action, retest result, and finding state (`confirmed`, `needs_validation`, `rejected`).

No evidence of impact means no finding. Missing deployment fact means `needs_validation`, not confirmed vulnerability.
