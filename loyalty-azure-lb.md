# Skill: Azure Load Balancer Diagnosis

Diagnose connection-level errors (resets, timeouts, client-reported 5xx) against MobileAppService, fronted by an Azure Standard Load Balancer directly on IIS VMs — no Application Gateway/WAF in this path. For Application Gateway WAF errors on WebSite/ClubGrido, use `/loyalty-azure-waf` instead; that skill covers a structurally different (L7) component.

## Known Infrastructure

| Field | Value |
|---|---|
| Subscription | Smart IT - Grido (`0190fa7d-4ccf-4e3d-beb1-323b5780bfc8`) |
| Resource group | `DefaultGroup01` |
| Load Balancer | `SFCG-MOBI-LB` (SKU: **Standard**, Layer 4/TCP only — cannot generate HTTP status codes) |
| Public IP | `20.121.19.174` (static) — resource `SFCG-MOBI-LB-publicip`. This is the LB's IP, **not** a VM NIC's IP — confirmed 2026-08-21 (`GITIN-1909`) correcting an earlier doc error. |
| Backend pool | `SFCG-MOBI-LB-backendpool01` — `SFCG-MOBI-01` (NIC `sfcg-mobi-01421`) and `SFCG-MOBI-02` (NIC `sfcg-mobi-02639`), both confirmed live members |
| Live LB rule | `SFCG-MOBI-LB-lbrule02` — TCP 8043→8043, `idleTimeoutInMinutes=15` |
| Active health probe | `SFCG-MOBI-LB-probe02` — plain **TCP**, port 8043, 5s interval, `numberOfProbes=1` |
| Ingress path | Client → NSG `sfcgnetsec01` (named multi-IP allow-lists, not subnet/`*`-based — rules `Allow-SmartFran-MobileApp` (P111) and `Grido-Mobile-Allow` (P113, includes `172.191.0.208`, the exact source IP seen throughout `GITIN-1909`'s httperr.log analysis), both TCP 8043 → `192.168.50.111`/`.112`; confirmed 2026-08-21, full detail `docs/azure_nsg.md`) → `SFCG-MOBI-LB` → IIS. No WAF/Application Gateway in this path — `WAF_APPs` exists in the same RG but routes WebSite/ClubGrido only. A separate local doc (`~/Documentos/git/smartfran-documentacion/sml-sf-mobile.md`) labels `172.191.0.208` as "APIM Grido" (Grido's API Management egress) and tracks a "clean" target IP list per rule — **treat both as reported, not confirmed**: per the user (2026-08-21), that repo's currency relative to `bots/` is unverified, and it should not be assumed authoritative over live Azure state without independent confirmation. |

**Known tech debt (not a bug to fix reactively):** two additional probes exist on the LB — `SFCG-MOBI-LB-probe01` (HTTP, `/api/MobileAppService/CheckServiceStatus`, `numberOfProbes=2`) and `healthcheck.ashx` (HTTPS, `/healthcheck.ashx`) — but neither is attached to the live rule. The active probe is TCP-only, so it cannot distinguish "port open" from "app actually ready to serve HTTP." Confirmed effect: when `SFCG-MOBI-02` was cold-started reactively during `GITIN-1909` (2026-08-21), the TCP probe marked it healthy before IIS/WAS was stable, and the LB routed live traffic into the ~70s startup gap — producing real 503s and connection resets **on that occasion**. Do not re-diagnose this as new; it's documented and out of scope unless it recurs and someone decides to prioritize the probe swap.

## Central gotcha: a client-reported "503" is not proof a 503 was sent

Any HTTP client using connection pooling (`HttpClient`/`SocketsHttpHandler`, most load-test tools) will occasionally try to reuse a pooled connection the server already closed — the failure surfaces client-side as a raw `SocketException`/`IOException` ("connection forcibly closed by the remote host"), **before** any HTTP response is parsed. Load-test tools frequently bucket this as "503" in their own reporting even though no HTTP status line was ever sent. Standard LB (L4) and NSGs (packet filters) are both structurally incapable of generating an HTTP status code at all — only IIS itself can. Before accepting a reported "503" as a real server-side response:

1. Confirm every component between client and app for L7 capability (see Step 1).
2. Get an exhaustive count from IIS's own httperr.log for the exact window — not a sampled excerpt (see Step 3). A sample can miss or, just as easily, mislead by coincidence; only a full scan is a defensible answer.

## Step 1 — Confirm LB config (don't trust the table above — re-verify)

```bash
SUB=0190fa7d-4ccf-4e3d-beb1-323b5780bfc8
RG=DefaultGroup01
LB=SFCG-MOBI-LB

az network lb rule list --lb-name $LB --resource-group $RG --subscription $SUB \
  --query "[].{name:name, protocol:protocol, frontendPort:frontendPort, backendPort:backendPort, idleTimeoutInMinutes:idleTimeoutInMinutes, probe:probe.id}" -o table

az network lb probe list --lb-name $LB --resource-group $RG --subscription $SUB \
  --query "[].{name:name, protocol:protocol, port:port, intervalInSeconds:intervalInSeconds, numberOfProbes:numberOfProbes, requestPath:requestPath}" -o table

az network lb address-pool list --lb-name $LB --resource-group $RG --subscription $SUB \
  --query "[].{name:name, backendIps:backendIPConfigurations[].id}" -o json
```

Also check for any Application Gateway/Front Door in the RG that might also terminate this traffic — don't assume the path is LB-only just because that's what's documented:

```bash
az resource list --resource-group $RG --subscription $SUB \
  --query "[?type=='Microsoft.Network/applicationGateways' || type=='Microsoft.Network/frontdoors'].{name:name, type:type}" -o table
```

If any listener/rule on a returned Application Gateway routes the hostname being investigated, that gateway (L7) **can** generate its own 502/503 — re-scope the investigation accordingly instead of assuming the LB-only path.

## Step 2 — Backend health over the incident window

```bash
az monitor metrics list --resource "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Network/loadBalancers/$LB" \
  --metric "DipAvailability" --interval PT1M \
  --start-time <START_UTC> --end-time <END_UTC> \
  --subscription $SUB -o table
```

A dip below 100% with `numberOfProbes=1` and a 2-VM pool is consistent with one backend missing a single probe cycle — correlate against VM boot/restart timing (Step 4) before assuming a load-driven flap.

## Step 3 — Exhaustive httperr.log scan (per VM — RDP required, PowerShell)

Run on **every** VM in the backend pool for the exact window — don't rely on the most recently-modified log file only; http.sys rotates, and a window can span files.

```powershell
$env:COMPUTERNAME   # confirm host before trusting anything below

$startWindow = Get-Date '<START_UTC>'   # e.g. '2026-08-21T11:00:00Z'
$endWindow   = Get-Date '<END_UTC>'

$inWindow = Get-ChildItem 'C:\Windows\System32\LogFiles\HTTPERR\httperr*.log' | Get-Content | ForEach-Object {
    $f = $_ -split '\s+'
    if ($f.Count -ge 11) {
        try { $ts = [datetime]::Parse("$($f[0])T$($f[1])Z").ToUniversalTime() } catch { return }
        if ($ts -ge $startWindow -and $ts -lt $endWindow) { [PSCustomObject]@{ Line=$_; Status=$f[10]; Ts=$ts } }
    }
}

"Total httperr.log entries in window: $($inWindow.Count)"
"Entries with sc-status=503:          $(($inWindow | Where-Object { $_.Status -eq '503' }).Count)"
$inWindow | Where-Object { $_.Status -eq '503' } | Select-Object -ExpandProperty Line
```

Field order (http.sys error log): `date time c-ip c-port s-ip s-port cs-version cs-method cs-uri cs-username sc-status s-siteid s-reason s-queue-name` — index 10 (0-based) is `sc-status`. `Timer_ConnectionIdle` in the `s-reason` field (index 12) is http.sys closing a keep-alive connection past `connectionTimeout` — a pure connection-level reset, no HTTP response at all; distinct from a real `sc-status=503` line.

## Step 4 — Check `connectionTimeout` and app-pool recycling on each VM

```powershell
$env:COMPUTERNAME

# IIS idle-connection timeout — check this before assuming a connection reset is unexplained.
# Confirmed 2026-08-21: MobileAppService runs connectionTimeout=15s (both VMs) — deliberate
# security hardening (anti connection-exhaustion), not misconfiguration. 8x more aggressive
# than IIS's 120s default. A pooled client (like most load-test tools) with a longer idle-reuse
# assumption than 15s will routinely race this and see "connection forcibly closed by remote host."
Get-ItemProperty "IIS:\Sites\SmartLoyalty.MobileAppService" -Name limits.connectionTimeout | Select-Object Value

# App pool recycling / rapid-fail-protection — rules in or out a WAS-driven cause
Get-WinEvent -FilterHashtable @{
  LogName = 'System'
  ProviderName = 'Microsoft-Windows-WAS'
  StartTime = (Get-Date '<START_UTC>')
  EndTime = (Get-Date '<END_UTC>')
} -ErrorAction SilentlyContinue | Select-Object TimeCreated, Id, Message
```

## Step 5 — Sequencing discipline

If any VM in the pool was started/restarted during the incident window (reactive scale-out, crash, maintenance), **establish precise before/after ordering relative to that event before attributing any symptom to it.** A VM coming online can produce its own genuine 503s/resets during its startup gap (Step 1's probe-threshold gotcha) — those are real, but if the original report predates the restart, they cannot be its explanation. Confirmed pattern (`GITIN-1909`, 2026-08-21): a reactive scale-out to `SFCG-MOBI-02`, taken *in response to* an already-reported incident, produced its own distinct 503 burst that had to be explicitly separated from the original cause once sequencing was clarified — don't let a later, self-inflicted event get conflated with what triggered the report in the first place.

## Constraints

- Default to read-only commands (`show`, `list`, metrics queries, `Get-WinEvent`, log reads).
- Any LB config change (probe swap, rule edit, backend pool membership) is a separate, explicit action outside this skill's scope, and follows the root CLAUDE.md destructive-command banner if it touches a live resource.

## Output

Findings go to `loyalty/events/YYYYMMDD_<description>/` per `loyalty-sre-output` — `investigation.md` first, `ops.md`/`ops-events.md` once converged. Full worked example: `events/20260821_mobileappservice_loadtest_503/`.
