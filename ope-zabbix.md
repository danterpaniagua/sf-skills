# Skill: Zabbix Custom Healthcheck Integration

Wire a custom bash healthcheck script into Zabbix: UserParameter, master + dependent items, macro-driven thresholds, triggers, and alert routing. Use whenever a new external dependency (third-party API, internal service) needs proactive monitoring beyond simple reachability.

## Scope

- Custom UserParameter items backed by a bash script (not native Zabbix templates)
- Dependent items with JSONPath preprocessing
- Host macros for trigger thresholds (never hardcode numbers in trigger expressions)
- Trigger dependencies to avoid duplicate alerts
- Tagging for existing alert-action routing (Google Chat, etc.)

Not in scope: Zabbix server administration (upgrades, HA, DB maintenance), native template-based monitoring (SNMP, native agent checks) — use vendor templates for those instead of a custom script.

## Known Infrastructure

| Resource | Value |
|---|---|
| Zabbix host | `Zabbix server` (FQDN `sf-monitoreo.smartfran.com`) |
| Version | Zabbix 6.0.32 (agent2) |
| Agent config dir | `/etc/zabbix/zabbix_agent2.d/` |
| Script location convention | `/opt/scripts/<name>.sh`, symlinked from `/usr/bin/<name>` |
| Severity naming | Classic 6.0: Not classified / Information / Warning / Average / High / Disaster |

## Command Constraint

> **Read commands** (`zabbix_agent2 -t`, `journalctl`, `cat`, `namei -l`) — safe to run, no warning needed.
> **Write/action commands** (`systemctl restart`, `mv`, `ln -s`, `chmod`) — always prefix the block with `⚠️ ACTION — this command modifies server/agent state.`

---

## Script contract

The healthcheck script must support a `--json` mode returning:

```json
{"overall_status":0,"checks":[{"service":"<name>","detail":"<k=v,...>","status":"UP|DOWN|DEGRADED","latency_ms":<int>}]}
```

`overall_status`: `0` = OK, `1` = DOWN, `2` = DEGRADED. Keep the script's own DEGRADED threshold as a rough internal signal only — real alerting thresholds belong in Zabbix macros (see below), not hardcoded in the script.

---

## Step 1 — Script placement

```bash
# ⚠️ ACTION — creates/moves files, changes permissions
sudo mkdir -p /opt/scripts
sudo mv <script>.sh /opt/scripts/<script>.sh
sudo chmod 755 /opt/scripts/<script>.sh
sudo ln -s /opt/scripts/<script>.sh /usr/bin/<script>
```

**Gotcha:** never leave the script under a personal home directory (e.g. `/home/ubuntu/...`). Home directories are commonly `750`, which blocks the `zabbix` user regardless of the script file's own permissions. `/opt/scripts` avoids this by construction.

## Step 2 — UserParameter

```bash
# /etc/zabbix/zabbix_agent2.d/userparameter_<name>.conf
UserParameter=<name>.status.raw,/usr/bin/<script> --json
```

```bash
# ⚠️ ACTION — restarts the Zabbix agent
sudo systemctl restart zabbix-agent2
```

**Gotcha:** the agent only reads `.conf` files on start. Creating or editing a UserParameter file without restarting produces `ZBX_NOTSUPPORTED [Unknown metric]` on test — confirm via `journalctl -u zabbix-agent2` timestamps if this happens.

```bash
# Verify the key resolves
sudo -u zabbix zabbix_agent2 -t <name>.status.raw
```

## Step 3 — Master item

`Data collection → Hosts → <host> → Items → Create item`

| Field | Value |
|---|---|
| Name | `<Name> Raw Status` |
| Type | Zabbix agent |
| Key | `<name>.status.raw` |
| Type of information | Text |
| Update interval | `{$<NAME>.UPDATE.INTERVAL}` |

Route the interval through a macro rather than hardcoding it — `{$<NAME>.NODATA.WINDOW}` (Step 5) is derived proportionally from it (~10x), so both stay in sync from one edit point.

## Step 4 — Dependent items

One per field of interest, Type = **Dependent item**, Master item = the item above. Add a **JSON Path** preprocessing step per item:

| Name | Key | Type of info | JSONPath |
|---|---|---|---|
| `<Name>` Overall Status | `<name>.status.overall` | Numeric (unsigned) | `$.overall_status` |
| `<Name>` `<Service>` Status | `<name>.status.<service>` | Text | `$.checks[?(@.service=='<SERVICE>')].status.first()` |
| `<Name>` `<Service>` Latency | `<name>.status.<service>.latency` | Numeric (unsigned) | `$.checks[?(@.service=='<SERVICE>')].latency_ms.first()` |

Test every JSONPath with the **Test** button using a captured real `--json` output before saving.

## Step 5 — Host macros for thresholds

Add on `Data collection → Hosts → <host> → Macros`. Never hardcode threshold numbers directly in trigger expressions — always route them through a macro so tuning is a config edit, not a trigger edit.

| Macro | Example value | Purpose |
|---|---|---|
| `{$<NAME>.UPDATE.INTERVAL}` | `1m` | Master item polling interval |
| `{$<NAME>.LATENCY.WARN}` | `2500` | Latency (ms) — Warning threshold |
| `{$<NAME>.LATENCY.HIGH}` | `5000` | Latency (ms) — Critical threshold |
| `{$<NAME>.LATENCY.WARN.RECOVERY}` | `2000` | Latency (ms) — recovery point for the Warning trigger (hysteresis) |
| `{$<NAME>.LATENCY.HIGH.RECOVERY}` | `4000` | Latency (ms) — recovery point for the Critical trigger (hysteresis) |
| `{$<NAME>.LATENCY.AVG.PERIOD}` | `5m` | Averaging window for latency triggers |
| `{$<NAME>.NODATA.WINDOW}` | `10m` | `nodata()` window (~10x update interval) |

Derive `LATENCY.WARN` / `LATENCY.HIGH` from real history, not guesswork: pull 30 days of the latency item's min/avg/max from **Monitoring → Latest data → Graph**, then set Warning just above the observed max and High at ~2x max. Set the `.RECOVERY` macros ~20% below their paired threshold — without hysteresis, an `avg()` value oscillating right at the threshold will open/close the same problem repeatedly.

## Step 6 — Triggers

`Data collection → Hosts → <host> → Triggers → Create trigger`. Give each trigger as its own block, not a wide table — expressions are long and a markdown table makes them impossible to copy-paste cleanly.

To surface *which* service is failing directly in the Problems list (not just "DOWN"/"DEGRADADO" with no detail), pull the per-service status items into the DOWN/DEGRADADO expressions with a no-op clause, then reference them via `{ITEM.LASTVALUE2}` / `{ITEM.LASTVALUE3}` (indices follow the order items appear in the expression) in Operational data:

```
Name: <Name> DOWN
Severity: High
Expression: last(/<host>/<name>.status.overall)=1 and last(/<host>/<name>.status.<serviceA>)<>"__unused__" and last(/<host>/<name>.status.<serviceB>)<>"__unused__"
OK event generation: Expression
Operational data: <ServiceA>: {ITEM.LASTVALUE2} | <ServiceB>: {ITEM.LASTVALUE3}
```

```
Name: <Name> DEGRADADO
Severity: Warning
Expression: last(/<host>/<name>.status.overall)=2 and last(/<host>/<name>.status.<serviceA>)<>"__unused__" and last(/<host>/<name>.status.<serviceB>)<>"__unused__"
OK event generation: Expression
Operational data: <ServiceA>: {ITEM.LASTVALUE2} | <ServiceB>: {ITEM.LASTVALUE3}
```

```
Name: <Name> sin datos
Severity: Average
Expression: nodata(/<host>/<name>.status.raw,{$<NAME>.NODATA.WINDOW})=1
OK event generation: Expression
Operational data: Sin datos hace más de {$<NAME>.NODATA.WINDOW}
```

```
Name: <Name> <Service> Latencia Alta
Severity: Warning
Expression: avg(/<host>/<name>.status.<service>.latency,{$<NAME>.LATENCY.AVG.PERIOD})>{$<NAME>.LATENCY.WARN}
OK event generation: Recovery expression
Recovery expression: avg(/<host>/<name>.status.<service>.latency,{$<NAME>.LATENCY.AVG.PERIOD})<{$<NAME>.LATENCY.WARN.RECOVERY}
Operational data: Latencia promedio: {ITEM.LASTVALUE1}ms (umbral: {$<NAME>.LATENCY.WARN}ms)
```

```
Name: <Name> <Service> Latencia Crítica
Severity: High
Expression: avg(/<host>/<name>.status.<service>.latency,{$<NAME>.LATENCY.AVG.PERIOD})>{$<NAME>.LATENCY.HIGH}
OK event generation: Recovery expression
Recovery expression: avg(/<host>/<name>.status.<service>.latency,{$<NAME>.LATENCY.AVG.PERIOD})<{$<NAME>.LATENCY.HIGH.RECOVERY}
Operational data: Latencia promedio: {ITEM.LASTVALUE1}ms (umbral: {$<NAME>.LATENCY.HIGH}ms)
```

**Trigger dependencies** (Triggers → `<trigger>` → Dependencies tab) — set the full chain, not just the obvious pair, to avoid stacked alerts for one root cause:

```
<Name> sin datos              → depends on: (none — top of the chain; pipeline itself broken)
<Name> DOWN                   → depends on: <Name> sin datos
<Name> DEGRADADO              → depends on: <Name> sin datos
<Name> <Service> Latencia Alta     → depends on: <Name> <Service> Latencia Crítica, <Name> DOWN, <Name> sin datos
<Name> <Service> Latencia Crítica  → depends on: <Name> DOWN, <Name> sin datos
```

Rationale: `sin datos` means the last known values are frozen/stale, so it masks everything else. A full outage's timeout can also spike latency, so both latency triggers depend on `DOWN`. `Latencia Crítica`'s threshold is strictly higher than `Latencia Alta`'s, so Crítica firing always implies Alta's condition is also true — Alta depends on Crítica to avoid double-alerting on the same spike. Do **not** make `DEGRADADO` depend on the latency triggers — the script's own DEGRADED threshold also covers non-WSFEv1 checks that Zabbix isn't independently tracking, so it isn't fully redundant with them.

## Step 7 — Alert routing

Before creating triggers, check `Alerts → Actions → Trigger actions` for an existing action that already covers the right notification channel, and read **what condition it matches on** (host group vs. tag) — don't assume group membership routes it.

If the action matches on a tag (common pattern: `service_group` = `<GROUP>`), add that exact tag to every new trigger's **Tags** tab. Adding the host to a group does nothing if the action filters on tags, and vice versa.

---

## Known gotchas (from prior integrations)

- **TLS handshake failures against legacy servers**: `curl: (35) ... dh key too small` — OpenSSL 3.x's default `SECLEVEL=2` rejects weak DHE parameters some older government/enterprise endpoints still negotiate. Fix: add `--ciphers 'DEFAULT@SECLEVEL=1'` to the curl call in the script. Not a sign the target is actually down — verify with the endpoint's own dummy/health method if one exists.
- **Script vs. Alerts → Scripts confusion**: a command loaded under `Alerts → Scripts` is manual/on-demand only — no history, no triggers. Continuous monitoring always needs an **item** backed by a **UserParameter**, not a Scripts entry.
- **`length()` trigger function doesn't exist before Zabbix 6.4**: pulling extra items into a trigger expression as a no-op (e.g. for Operational data) via `length(/host/key,#1)>=0` fails with `incorrect usage of function "length"` on 6.0/6.2 — the parser doesn't recognize it and misreports it as a parameter error. On Zabbix 6.0, use `last(/host/key)<>"__unused__"` instead (valid since early versions, no period argument needed, works on Text-type items).

## Constraints

- Default to read-only commands; prefix state-changing ones with `⚠️ ACTION`.
- Never restart the Zabbix server process itself or touch server-side config (`zabbix_server.conf`) without explicit request — only agent-side (`zabbix-agent2`) actions are in scope here.
- Output commands as copy-paste blocks. User runs them and pastes results back — never execute directly.
- Frontend steps (items/triggers/macros/actions) are manual-walkthrough field tables, not automatable via this skill unless the user explicitly asks for Zabbix API scripting.
