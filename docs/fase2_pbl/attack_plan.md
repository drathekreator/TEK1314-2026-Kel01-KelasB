# Attack Plan — Isolated VM Outage, Recovery & Patch Exercise

**Project:** TEK1314 Cybersecurity PBL, Kelompok 01  
**Scenario:** Red Team causes bounded service outages inside isolated lab VMs. Blue Team detects, restores services, applies hardening/security patches, then verifies recovery.  
**Scope:** `target-server` (`192.168.1.5`, `192.168.56.10`), `iot-node` (`192.168.1.40`), `cyber-workstation` (`192.168.1.20`), `sec-onion` (`192.168.56.30`), Docker stack `/opt/iot-target-stack`.  
**Excluded:** Stealth, SIEM evasion, log deletion, persistence, backdoors, data theft, external/NAT hosts, shared networks, permanent damage.

## 1. Objectives

- Validate detection of service loss and dependent-service failures.
- Measure detection time, recovery time, and restoration success against agreed RTO/RPO.
- Exercise VM snapshot restore and Blue Team recovery process.
- Apply security fixes after recovery and verify via repeat tests.

## 2. Safety Rules

1. Use isolated VirtualBox Internal Network and Host-Only only; disable bridged networking. Do not touch NAT/external addresses.
2. Snapshot `target-server`, `iot-node`, and `sec-onion`; verify restore access before tests.
3. Run one failure at a time; default injection window 60 seconds, hard maximum 120 seconds.
4. Keep target console access. Prefer reversible service stops; no resource exhaustion or data deletion.
5. Stop immediately on hypervisor instability, suspected data corruption, traffic leaving lab, SIEM overload, or an unexpected asset response.
6. Red Team injects fault. Blue Team restores service and applies remediation.
7. Record UTC start/end, test ID, command, observed impact, detection evidence, restore action, recovery time, and retest result.

## 3. Roles and Preflight

- **Lead:** Approves scope, window, duration, stop conditions, and RTO/RPO.
- **Red Team:** Executes only listed bounded fault injections and preserves evidence.
- **Blue Team:** Monitors Security Onion, declares detection, restores services, patches, and retests.
- Confirm snapshots, console access, no bridged NICs, synchronized clocks, and contact channel.
- Blue Team records baseline service health, Docker status, API `/health`, MQTT telemetry heartbeat, SSH access, UFW, SIEM sensor health, Zeek/Suricata state, and baseline alert count.
- If baseline traffic is not visible in Security Onion, pause fault injection until sensor limitation is documented.

## 4. Exercise Sequence

### Phase A — Baseline

Run `P-01` through `P-03` from `attack_command.md`. Confirm all expected services healthy and logs visible.

### Phase B — Fault injection

For each approved AV test:

1. Lead calls test ID and confirms Blue Team ready.
2. Record pre-state and UTC start.
3. Red Team runs only the listed reversible command.
4. Blue Team identifies fault from telemetry and records time-to-detect.
5. Restore at detection or at 60 seconds, whichever comes first; absolute stop at 120 seconds.
6. Blue Team runs health checks; record time-to-recover and service/data integrity.
7. Do not start next test until Lead confirms healthy baseline.

### Phase C — Patch and recovery hardening

Blue Team restores snapshot if needed, then applies documented changes. Likely fixes from blueprint: disable anonymous MQTT and add topic ACLs; protect `/api/telemetry` with authentication/authorization; add TLS if required; remove hard-coded DB credentials; add container healthchecks and downtime alerts; align firewall/interface rules; verify log forwarding and time sync. Update OS/packages through approved maintenance process.

### Phase D — Retest

Rerun only failed AV tests and relevant baseline checks. Confirm both service recovery and Security Onion detection. Record patch versions/config changes and attach before/after evidence.

## 5. Bounded Availability Test Cases

| ID | Fault | Expected signal | Approved restore | Maximum |
|---|---|---|---|---|
| AV-01 | Stop Nginx container | `/health` fails; Zeek HTTP gap; container state event | Start `iot-nginx` | 60 sec |
| AV-02 | Stop API container | Nginx returns upstream failure; API health fails | Start `iot-telemetry-api` | 60 sec |
| AV-03 | Stop Mosquitto container | Broker unavailable; telemetry heartbeat stops | Start `iot-mosquitto` | 60 sec |
| AV-04 | Stop PostgreSQL container | API logs DB connection failure; container event | Start `iot-postgres` | 60 sec |
| AV-05 | Stop IoT publisher service | Telemetry heartbeat stops | Start `iot-sensor` via OpenRC | 60 sec |
| AV-06 | Stop SSH daemon from target console | Management SSH fails; service state change logged | Start SSH service locally | 60 sec |
| AV-07 | Briefly block tester source to one approved service using temporary firewall rule | Connection denied; UFW event | Delete exact temporary rule | 30 sec |

AV-07 requires Lead approval and a second operator at target console. Save exact rule handle/rule number before insertion; remove only that rule. Do not flush/reset firewall. No CPU/memory/disk/network flood tests. No sensor shutdown test in this run; validate sensor-health monitoring by reviewing status and existing logs instead.

## 6. Patch / Recovery Checklist

- Confirm Docker `restart` policy and healthchecks for Mosquitto, API, Nginx, PostgreSQL.
- Alert on container stop/restart, endpoint health failure, MQTT heartbeat absence, and sensor/log pipeline loss.
- Disable anonymous MQTT; apply least-privilege topic ACLs and authenticated publisher.
- Add appropriate auth to telemetry API; keep `/health` minimal and non-sensitive.
- Remove static credentials from source/Compose; rotate credentials through approved secret management.
- Define HTTP/HTTPS policy; enable TLS if data/auth crosses networks requiring confidentiality.
- Verify PostgreSQL remains Docker-network-only; confirm published ports bind only intended interface.
- Verify UFW rules, SSH management restriction, Auditd, Fail2ban, patch status, and time sync.

## 7. Evidence and Completion

For each test record: run ID, operator, UTC start/end, VM snapshot ID, test ID, exact command, pre/post health, detection timestamp, event IDs, restore command, time-to-recover, data integrity check, patch details, retest result, and sign-off.

Complete when every approved fault was restored within agreed RTO, telemetry and logs are intact, patches are applied, retests pass, and no temporary rule/service/account remains. This tests selected failure modes; it does not prove resilience against every outage cause.

## 8. Blueprint Risks (Not Yet Confirmed Runtime Findings)

The blueprint shows anonymous Mosquitto (`allow_anonymous true`), unauthenticated `/api/telemetry`, HTTP-only Nginx, static database credentials, multi-interface exposure, and SSH allowed only from management subnet while attacker VM is on internal subnet. Treat these as source-level risks; verify configuration and intended policy before declaring findings.

## Test IDs

- `P-01` baseline and VM/network verification
- `P-02` service and telemetry health
- `P-03` SIEM readiness
- `AV-01` to `AV-07` bounded availability fault cases
- `P-04` recovery, patch verification, cleanup, and sign-off

No command list covers every vulnerability or failure mode. Add cases only through approved scope change and risk review.
