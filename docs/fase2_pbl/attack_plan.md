# Attack Plan — Blue Team Control Validation

**Project:** TEK1314 Cybersecurity PBL, Kelompok 01  
**Purpose:** Validate documented hardening and SIEM coverage with safe, attributable tests.  
**Scope:** `target-server` (`192.168.1.5`, `192.168.56.10`, NAT DHCP); `iot-node` (`192.168.1.40`); `cyber-workstation` (`192.168.1.20`); `sec-onion` (`192.168.56.30`). Test only during approved lab window.  
**Excluded:** Stealth, SIEM evasion, brute force, exploitation, arbitrary command execution, privilege escalation, persistence, destructive traffic, and extraction or alteration of real telemetry/data.

## 1. Key Blueprint Risks to Validate

These are source-level concerns from the blueprint, not confirmed runtime findings:

1. Mosquitto config sets `allow_anonymous true` and listens on `0.0.0.0:1883`. Anonymous access may allow unauthorized telemetry read/write unless other controls intervene.
2. Nginx exposes `/api/telemetry`; API route returns the latest 20 records without visible authentication or authorization. Sensitive data exposure depends on deployment and data classification.
3. Nginx uses HTTP on port 80 only; TLS is not configured in shown config. Credentials/session confidentiality risk depends on network and application use.
4. Docker Compose maps Mosquitto and Nginx ports to all host interfaces by default; target also has internal, management, and NAT interfaces. Effective bind/firewall behavior requires runtime validation.
5. PostgreSQL has no host port mapping in shown Compose, which is positive, but application config contains default credentials and Compose has fixed database credentials. Validate secret handling and ensure no other exposure.
6. SSH is allowed by UFW only from `192.168.56.0/24`, while attacker workstation resides on `192.168.1.0/24`. Test expected denial from workstation; authorized management access is a separate Blue Team check.
7. UFW rules shown allow HTTP and MQTT from internal subnet, but not HTTPS; IP plan mentions 443 although blueprint has no TLS listener. Confirm intended state.
8. `net.ipv4.ip_forward=0` and NAT interface may conflict with routing expectations. Verify routing; do not change it during assessment.
9. Node API accepts JSON and inserts MQTT-derived fields into database. Parameterized SQL is used, but input validation and topic authorization are not shown. Use only safe test namespace/principal; do not inject payloads.
10. IoT publisher uses unauthenticated MQTT client in shown code and publishes to production telemetry topic. Do not alter or spoof that topic; validate publisher identity through config review.

## 2. Rules of Engagement

- Use only listed lab nodes and explicitly allowed ports. Do not test NAT addresses or any external host.
- Obtain Blue Team approval before active checks; snapshot VMs and synchronize clocks.
- Use disposable credentials and synthetic data. Never publish to `iot/sensor/telemetry` during Red Team tests.
- Low-rate, single-target checks only. Stop on unexpected response, instability, out-of-scope route, real-data exposure, or sensor overload.
- Preserve timestamps, source/destination, test ID, exact command, output, host log evidence, and SIEM event IDs.
- Red Team does not modify configuration. Blue Team performs privileged checks, cleanup, and remediation.

## 3. Procedure and Expected Outcomes

### Phase 0 — Authorization and sensor baseline

Confirm test window, assets, approved ports, test accounts/topics, and Blue Team contact. Blue Team confirms Security Onion sensor state, SPAN capture on `eth0`, management access on `192.168.56.30`, active Suricata rules, Zeek logging, time synchronization, and baseline event count. If packet visibility is absent, pause tests and record a monitoring limitation.

### Phase 1 — Network exposure and interface policy

From `cyber-workstation`, check only TCP `22, 80, 443, 1883, 5432` on `192.168.1.5`. Blue Team separately reviews listeners and firewall rules on all target interfaces, including Host-Only and NAT. Compare observed exposure with intended policy:

- SSH: management subnet only; internal workstation expected denied.
- HTTP: internal lab only, if required.
- HTTPS: not configured in shown blueprint; confirm expected status before labeling.
- MQTT: internal lab only; restrict to intended publisher/API where possible.
- PostgreSQL: no host exposure; do not connect or authenticate.

Unexpected open port or interface binding becomes a finding; do not enumerate further without scope approval.

### Phase 2 — SSH access controls

Blue Team reviews effective SSH configuration and firewall. From internal workstation, make at most one connection attempt to port 22; expected result is blocked if policy is applied. From an authorized management client only, perform one failed login using disposable account then one successful approved account login. Check root login disabled, `MaxAuthTries=3`, `LoginGraceTime=30`, X11 and TCP forwarding disabled, session logging present, and Fail2ban behavior using Blue Team evidence. Do not deliberately trigger bans through repeated attempts.

### Phase 3 — Web and API authorization / transport

Use read-only HTTP requests to `/health` and `/api/telemetry`. Test only synthetic lab data; Blue Team confirms response is not sensitive. Record authentication behavior, status code, headers, content type, and Nginx/API logs. Confirm whether API should require authentication and whether HTTPS is required. Do not submit malformed input, modify records, or test injection.

### Phase 4 — MQTT access-control validation

Blue Team reviews effective Mosquitto config and ACLs. If approved, use disposable client against a dedicated `lab/test/<run-id>` topic only. Validate one authorized operation and one denied operation with explicitly provisioned test principals. Confirm anonymous access status from config and broker logs. Never subscribe or publish to `iot/sensor/telemetry`, wildcard topics, or other clients' topics; never test retained-message or wildcard exposure on real topics.

### Phase 5 — Container/database boundary

Blue Team inspects Compose rendering, port bindings, container network, privileges, mounts, environment variables, logs, and secret source. Confirm PostgreSQL has no host binding and is reachable only by API service. Do not query DB or attempt credentials. Flag default/static credentials in source/config for rotation and secret-store remediation without attempting authentication.

### Phase 6 — Host hardening and telemetry

Blue Team checks UFW/nftables, interface routes, IP forwarding, sysctl values, package updates, Fail2ban, Auditd rules, service state, and log permissions. Correlate each permitted test to Zeek `conn.log`/`http.log`/MQTT records, Suricata alerts if applicable, Nginx/Mosquitto logs, and SSH auth logs. Missing event is a monitoring gap, not proof of evasion.

### Phase 7 — Findings, remediation, retest

Document asset, control, test ID, exact evidence, demonstrated impact, severity, owner, minimal fix, and retest. Blue Team remediates. Red Team repeats only failed, bounded checks after approval. Remove test principal/data and any temporary ban; retain evidence per project policy.

## 4. Pass Criteria

- Services and interface bindings match approved policy.
- Internal SSH denied; management SSH follows explicit account and authentication policy.
- No unauthenticated API response exposes non-public data; transport policy is explicit.
- MQTT permissions are least privilege; anonymous access is disabled unless an explicit safe design justifies it.
- PostgreSQL is not host-exposed; default credentials are removed from deployable configuration.
- Required host/network events are timestamped, attributable, and searchable in Security Onion.
- All findings have owner and retest evidence. Scope is partial: this validates documented controls, not every possible vulnerability.

## 5. Source / Design Mismatches

- `ip_plan.md` lists `192.168.56.10` management IP and documents HTTPS `443`; hardening plan excerpts do not specify 443 rule/listener.
- Blueprint describes target NAT interface, but other plans do not consistently include it. Confirm interface/routing configuration before tests; never probe its DHCP address.
- Hardening plan says PostgreSQL restricted to Docker network; Compose supports that by omitting port mapping, but verify effective runtime mapping.
- Blueprint uses anonymous Mosquitto and no API auth in shown snippets, conflicting with least-privilege/security objectives. Runtime behavior remains unverified.

## Test Index

| ID | Control |
|---|---|
| T-01 | Lab baseline and sensor visibility |
| T-02 | Approved-port exposure on internal interface |
| T-03 | Blue Team listener, firewall, and interface review |
| T-04 | SSH policy and controlled auth evidence |
| T-05 | Read-only `/health` and `/api/telemetry` exposure check |
| T-06 | MQTT config review and isolated test-topic ACL check |
| T-07 | Docker/PostgreSQL exposure and secret configuration review |
| T-08 | Sysctl, forwarding, Fail2ban, Auditd, patch-state review |
| T-09 | SIEM correlation and monitoring-gap review |
| T-10 | Cleanup and retest |

**No command list can cover every possible vulnerability.** This plan targets controls and risks explicitly documented in the supplied blueprint; code review, dependency review, and additional authorized assessment require separate scope and evidence.
