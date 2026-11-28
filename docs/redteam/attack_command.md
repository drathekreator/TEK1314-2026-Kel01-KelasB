# Attack Command — Exploratory Red-Team Validation

**Scope:** `192.168.1.5` / `192.168.56.10` only, from approved lab workstation.
**Variables:**

```bash
export TARGET=192.168.1.5
export MGMT=192.168.56.10
export HOST=dreamsheep.local
export RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "evidence/$RUN_ID"
script -q -a "evidence/$RUN_ID/session.txt"
```

Use one block at a time. Keep request count low. Replace paths only after confirming deployment. Commands are discovery/validation, not exploitation.

## P-01 — Scope and route sanity

```bash
date -u +%FT%TZ
hostname
ip -br addr
ip route
ip route get "$TARGET"
ip route get "$MGMT"
```

```bash
# Stop if route uses NAT, bridged, or unexpected interface.
ss -Htn state established
```

## P-02 — Low-noise service discovery

```bash
nmap -Pn -n --reason --top-ports 100 --max-retries 1 --host-timeout 30s "$TARGET"
nmap -Pn -n -sV --version-light -p 22,80,443,1883,5432 "$TARGET"
```

```bash
for p in 22 80 443 1883 5432; do
  timeout 3 bash -c "</dev/tcp/$TARGET/$p" >/dev/null 2>&1 && echo "OPEN $p" || echo "CLOSED/FILTERED $p"
done
```

Expected blueprint result: `22` reachable only from management, web/MQTT from internal subnet, `5432` not externally reachable.

## P-03 — HTTP route and disclosure baseline

```bash
for path in / /health /api/readings /api/telemetry /robots.txt /.env; do
  echo "===== $path ====="
  curl --noproxy '*' -ksS --max-time 5 -D - -o "evidence/$RUN_ID$(echo "$path" | tr '/.' '__').body" "https://$HOST$path" | sed -n '1,25p'
done
```

```bash
curl --noproxy '*' -sS --max-time 5 -D - -o /dev/null "http://$HOST/health"
curl --noproxy '*' -ksS --max-time 5 "https://$HOST/health"
curl --noproxy '*' -ksS --max-time 5 "https://$HOST/api/readings" | head -c 4096
```

Do not treat `200 /health` as auth bypass. Test whether telemetry data is returned without session.

## RT-01/RT-02 — MQTT anonymous access and exposure

Read-only probe first:

```bash
timeout 8 mosquitto_sub -h "$TARGET" -p 1883 -t 'dreamsheep/telemetry' -C 1 -W 5 -v
```

```bash
# Topic discovery is bounded; no wildcard production subscription.
for topic in dreamsheep/telemetry dreamsheep/test lab/dreamsheep/telemetry; do
  echo "===== $topic ====="
  timeout 5 mosquitto_sub -h "$TARGET" -p 1883 -t "$topic" -C 1 -W 3 -v || true
done
```

Only with Lead approval, one benign marker:

```bash
printf '%s\n' '{"source":"red-team-validation","marker":"RT-MQTT-001","temperature":21.5,"humidity":50,"thi":60,"ammonia":0}' \
 | timeout 5 mosquitto_pub -h "$TARGET" -p 1883 -t 'dreamsheep/telemetry' -l -q 0
```

Observe one bounded result:

```bash
curl --noproxy '*' -ksS --max-time 5 "https://$HOST/api/readings" | grep -F 'RT-MQTT-001' || true
```

After evidence, do not publish more markers. Anonymous `CONNACK` alone proves access weakness, not data impact; pair with unauthorized subscribe/publish result.

## RT-03/RT-04 — API authorization, methods, and schema behavior

```bash
for path in /api/readings /api/telemetry /health; do
  for method in GET HEAD OPTIONS; do
    echo "===== $method $path ====="
    curl --noproxy '*' -ksS --max-time 5 -X "$method" -D - "https://$HOST$path" -o /dev/null | sed -n '1,15p'
  done
done
```

```bash
# Benign malformed/read-only query probes. No SQL/XSS payloads.
for q in '?limit=1' '?limit=0' '?limit=-1' '?limit=abc' '?format=json'; do
  echo "===== $q ====="
  curl --noproxy '*' -ksS --max-time 5 -D - "https://$HOST/api/readings$q" -o /dev/null | sed -n '1,15p'
done
```

Confirm whether response contains sensor data, internal errors, stack traces, or arbitrary marker. Stop at first state change.

## RT-05 — Configuration exposure and credential hygiene

From attacker, check only web exposure:

```bash
for path in /.env /docker-compose.yml /package.json /app/index.js /server.js /nginx/default.conf; do
  body="evidence/$RUN_ID/config$(echo "$path" | tr '/.' '__').body"
  code=$(curl --noproxy '*' -ksS --max-time 5 -o "$body" -w '%{http_code}' "https://$HOST$path")
  bytes=$(wc -c <"$body")
  printf '%-28s %s %s bytes\n' "$path" "$code" "$bytes"
done
```

Blue Team console validation (not attacker action):

```bash
sudo grep -RInE 'POSTGRES_PASSWORD|DATABASE_URL|MQTT_URL|allow_anonymous|serviceAccountKey' /opt/iot-target-stack --exclude-dir=node_modules --exclude='*.lock'
docker compose config --no-interpolate | sed -n '/environment:/,/^[^ ]/p'
```

Never print or copy real secrets into evidence. Record path and key name only.

## RT-06 — Client-side authentication boundary

```bash
curl --noproxy '*' -ksS --max-time 5 "https://$HOST/" -o "evidence/$RUN_ID/index.html"
grep -Eo '<script[^>]+src="[^"]+' "evidence/$RUN_ID/index.html" | cut -d'"' -f2
```

```bash
# Inspect downloaded frontend only for hardcoded demo auth; do not guess passwords online.
grep -RInE 'admin123|ternak123|ACCOUNTS|onLogin|localStorage|sessionStorage' "evidence/$RUN_ID" || true
```

Finding requires proof that dashboard access is granted solely client-side and protects a real sensitive action/data set.

## RT-07/RT-08 — Firebase validation

**Out of scope for this exercise:** Firebase is an external service and is not tested under the internal-only engagement. Treat Firebase observations as source-level hypotheses only. Do not send requests to the Firebase project from this run.

## RT-09 — TLS, redirect, certificate, and protocol checks

```bash
curl --noproxy '*' -sS -D - -o /dev/null "http://$HOST/health"
openssl s_client -connect "$TARGET:443" -servername "$HOST" -tls1_2 </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
for proto in -tls1 -tls1_1 -tls1_2 -tls1_3; do
  echo "===== $proto ====="
  timeout 5 openssl s_client "$proto" -connect "$TARGET:443" -servername "$HOST" </dev/null 2>&1 | grep -E 'Protocol|Cipher|handshake failure|alert protocol' | head
 done
```

Expected: TLS 1.2/1.3 accepted, older protocols rejected. Self-signed cert is availability/trust issue unless clients skip verification.

## RT-10 — Host-header routing behavior

```bash
for host in "$HOST" localhost 127.0.0.1 invalid.invalid; do
  echo "===== Host: $host ====="
  curl --noproxy '*' -ksS --max-time 5 -H "Host: $host" -D - https://"$TARGET"/health -o /dev/null | sed -n '1,15p'
done
```

No cache poisoning or external redirect testing. Report only if response changes security behavior or leaks data.

## RT-11/RT-12 — Runtime hardening verification

Blue Team console:

```bash
docker compose ps
docker inspect --format '{{.Name}} restart={{json .HostConfig.RestartPolicy}} health={{json .Config.Healthcheck}} ports={{json .HostConfig.PortBindings}}' \
  iot-mosquitto iot-postgres iot-telemetry-api iot-nginx
sudo ss -lntup
sudo ufw status numbered
sudo fail2ban-client status sshd 2>/dev/null || true
sudo sysctl net.ipv4.ip_forward net.ipv4.conf.all.rp_filter net.ipv4.conf.all.accept_redirects
```

Expected: PostgreSQL absent from host bindings; MQTT/web bindings match intended interface; healthchecks and restart policy visible; firewall source restrictions match blueprint.

## Cleanup and evidence close

```bash
date -u +%FT%TZ
# Confirm no temporary firewall rule or service state was changed by this run.
ss -lntup
```

Close `script` capture after testing:

```bash
exit
```

Do not delete logs. Remove only local evidence after report retention decision. Mark each hypothesis `confirmed`, `needs_validation`, or `rejected`; do not infer impact from banner/version alone.
