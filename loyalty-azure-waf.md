# Skill: Azure WAF Analysis

Diagnose Application Gateway WAF errors (504/502/403) reported against SmartLoyalty-hosted sites (ClubSite, ClubSite Management) — distinguish backend timeouts, backend unavailability, and genuine WAF blocks using Log Analytics access/firewall logs.

## Known Infrastructure

| Field | Value |
|---|---|
| Subscription | Smart IT - Grido (`0190fa7d-4ccf-4e3d-beb1-323b5780bfc8`) |
| Resource group | `DefaultGroup01` |
| Application Gateway | `WAF_APPs` (WAF_v2 tier) |
| Frontend public IP | `13.82.133.54` (`WAF_PROD01`) |
| Log Analytics workspace | `analisis-loadbalancer` (RG `defaultgroup01`) — diagnostic setting `LOGS_WAF` |
| WAF policy | `WAV_directiva` |

### Listener → backend pool map (confirmed 2026-07-21)

| Listener | Host | Frontend port | Backend pool | Backend settings | Timeout | Servers |
|---|---|---|---|---|---|---|
| `Listener_ClubSite_HTTPS` | `www.clubgrido.com.ar` | 443 | `Back_ClubSite` | `Backend_ClubSite` | 60s | `192.168.50.121`, `.122` |
| `Listener_ClubSite_HTTPS_PY` | `www.clubgrido.com.py` | 443 | `Back_ClubSite_PY` | `Backend_ClubSite_PY` | 60s | `192.168.50.121`, `.122` |
| `Listener_WebSite_HTTPS` | `gestion.clubgrido.com.ar` | 4430 | `Back_WebSite` | `Backend_WebSite` | 180s | `192.168.50.131` **(single server — no redundancy)** |

Re-verify this table with Step 1 before trusting it — backend pools/listeners can change.

## Command Constraint

Only read commands (`show`, `list`, `show-backend-health`, `az monitor log-analytics query`) — this skill is diagnostic only. Any config change (timeout, probe, pool membership) is a separate, explicit action outside this skill's scope, and follows the root CLAUDE.md destructive-command banner if it touches a live resource.

**Always pass `--subscription 0190fa7d-4ccf-4e3d-beb1-323b5780bfc8` explicitly on every `az` command** — never rely on `az account set`.

---

## Step 1 — Confirm current gateway config

Don't skip this even if the table above looks current — backend pools and routing rules change.

```bash
az network application-gateway show \
  --name WAF_APPs \
  --resource-group DefaultGroup01 \
  --subscription 0190fa7d-4ccf-4e3d-beb1-323b5780bfc8 \
  -o json > /tmp/appgw_waf_apps.json

jq '{
  backendHttpSettings: (.backendHttpSettingsCollection // [] | map({name, port, protocol, requestTimeout, probeEnabled: (.probe != null)})),
  probes: (.probes // [] | map({name, protocol, path, interval, timeout, unhealthyThreshold})),
  listeners: (.httpListeners // [] | map({name, hostName, frontendPort: .frontendPort.id, protocol})),
  routingRules: (.requestRoutingRules // [] | map({name, listener: .httpListener.id, backendPool: .backendAddressPool.id, backendSettings: .backendHttpSettings.id})),
  backendAddressPools: (.backendAddressPools // [] | map({name, backendAddresses}))
}' /tmp/appgw_waf_apps.json
```

**Gotcha:** no `urlPathMaps` means routing is purely by hostname (multi-site listener), not by URL path — a reported path like `/Catalog/Crud` is a page route inside the app, not necessarily the literal backend request URI. Check actual `requestUri_s` values in the access log (Step 4) rather than assuming the reported path is what was logged.

## Step 2 — Backend health

```bash
az network application-gateway show-backend-health \
  --name WAF_APPs \
  --resource-group DefaultGroup01 \
  --subscription 0190fa7d-4ccf-4e3d-beb1-323b5780bfc8 \
  -o json | jq '.backendAddressPools[] | {pool: .backendAddressPool.id, servers: [.backendHttpSettingsCollection[].servers[] | {address, health, healthProbeLog}]}'
```

**Gotcha:** `Healthy` here only proves the default probe's target (root `/`, unless a custom probe is configured — check `probes` from Step 1) responds. It does **not** prove a specific deep endpoint (e.g. `/Catalog/SetListCatalog`) is functional. A healthy probe + a real 504 on one endpoint is a strong signal of an application-level hang on that specific route, not an infra/network problem.

## Step 3 — Confirm Log Analytics workspace + get its GUID

```bash
az monitor diagnostic-settings list \
  --resource "$(az network application-gateway show --name WAF_APPs --resource-group DefaultGroup01 --subscription 0190fa7d-4ccf-4e3d-beb1-323b5780bfc8 --query id -o tsv)" \
  --subscription 0190fa7d-4ccf-4e3d-beb1-323b5780bfc8 \
  -o table

az monitor log-analytics workspace show \
  --resource-group defaultgroup01 \
  --workspace-name analisis-loadbalancer \
  --subscription 0190fa7d-4ccf-4e3d-beb1-323b5780bfc8 \
  --query customerId -o tsv
```

## Step 4 — Access log query

**Gotcha:** `AzureDiagnostics` type-suffixes every column (`_s`, `_d`, `_g`, `_t`) based on inferred type, and the suffix is **not guessable** — `timeTaken` is `timeTaken_d` (numeric), not `timeTaken_s`. Always run a schema probe before writing the real query:

```bash
az monitor log-analytics query \
  --workspace <WORKSPACE_GUID> \
  --subscription 0190fa7d-4ccf-4e3d-beb1-323b5780bfc8 \
  --analytics-query "AzureDiagnostics
| where Category == 'ApplicationGatewayAccessLog'
| where TimeGenerated > ago(48h)
| take 1" \
  -o json
```

Then the real query — filter by hostname, not by the reported path (see Step 1 gotcha), and always project `error_info_s` and `serverStatus_s`, not just `httpStatus_d`:

```bash
az monitor log-analytics query \
  --workspace <WORKSPACE_GUID> \
  --subscription 0190fa7d-4ccf-4e3d-beb1-323b5780bfc8 \
  --analytics-query "AzureDiagnostics
| where Category == 'ApplicationGatewayAccessLog'
| where TimeGenerated > ago(48h)
| where host_s has '<hostname>'
| project TimeGenerated, host_s, requestUri_s, httpMethod_s, httpStatus_d, serverStatus_s, error_info_s, timeTaken_d, serverConnectTime_s, serverHeaderTime_s, serverResponseLatency_s, backendPoolName_s, backendSettingName_s, serverRouted_s, clientIP_s
| order by TimeGenerated desc" \
  -o table
```

For large result sets, redirect to a file and analyze offline (`python3` with `re.split(r' {2,}', line.strip())` on the `-o table` output — az's table formatter pads columns with 2+ spaces, but request URIs/filenames can contain single embedded spaces, so split only on runs of 2+). Don't try to eyeball thousands of rows in the terminal.

### `error_info_s` reference — what each value actually means

| Value | Meaning | Points to |
|---|---|---|
| `ERRORINFO_UPSTREAM_TIMED_OUT` | Gateway waited the full configured `requestTimeout` and the backend never responded | App-level hang on that specific route (slow query, lock, stalled dependency) — not a gateway/WAF problem |
| `ERRORINFO_UPSTREAM_NO_LIVE` | Zero live servers in the backend pool at request time | Backend pool fully down/unreachable — check server uptime, app pool crashes, reboot history. Usually far higher volume than timeout errors during an outage window |
| `ERRORINFO_CLIENT_CLOSED_REQUEST` | Client disconnected before the gateway finished (often shows as `499`) | Often a symptom of the same slow backend as `UPSTREAM_TIMED_OUT` — users abandoning a hung request before it reaches the full timeout. Cross-check `timeTaken_d` on these rows |
| `ERRORINFO_CLIENT_TIMED_OUT` | Client-side timeout | Usually not actionable server-side; check volume/pattern before dismissing |
| `ERRORINFO_NO_ERROR` | Normal request | — |

**Always check `timeTaken_d` against the exact configured `requestTimeout` from Step 1** — an exact match (to the millisecond, e.g. `180.01`s against a 180s setting) confirms the gateway is behaving correctly and the backend is the actual problem, ruling out gateway misconfiguration as the cause.

## Step 5 — Rule out (or confirm) the WAF itself

Check `WAFMode_s` in the Step 4 schema probe first — if it's `Detection`, the WAF **cannot** be blocking anything (it only logs matches), which rules it out immediately without needing a firewall-log query at all.

If `WAFMode_s` is `Prevention`, or you need to confirm regardless:

```bash
az monitor log-analytics query \
  --workspace <WORKSPACE_GUID> \
  --subscription 0190fa7d-4ccf-4e3d-beb1-323b5780bfc8 \
  --analytics-query "AzureDiagnostics
| where Category == 'ApplicationGatewayFirewallLog'
| where TimeGenerated > ago(48h)
| where hostname_s has '<hostname>' or requestUri_s has '<path>'
| project TimeGenerated, hostname_s, requestUri_s, action_s, ruleGroup_s, ruleId_s, details_message_s
| order by TimeGenerated desc" \
  -o table
```

`action_s: "Matched"` means the rule fired but did **not** block (Detection mode, or an `Anomaly` scoring rule that didn't cross the block threshold in Prevention mode). Only `action_s: "Blocked"` is an actual WAF block.

## Step 6 — Performance log (v1 SKU only — usually skip for WAF_APPs)

`ApplicationGatewayPerformanceLog` does **not** populate for v2-tier gateways (`WAF_APPs` is v2) — backend health/throughput moved to Azure Monitor metrics instead. An empty result from this category against `WAF_APPs` is expected, not a data gap — don't spend time chasing it.

---

## Output

Findings go to `operations/events/YYYYMMDD_<description>/` per the root `operations/CLAUDE.md` convention (Resumen, Tabla resumen, Causa raíz, Hallazgos, Recursos afectados, Comandos ejecutados, Acciones propuestas) — this skill is diagnostic/read-only and does not write directly to `events/`.
