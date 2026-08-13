# Skill: Cloud Azure Operations

Investigation and remediation support for the SmartFran Cloud platform on Azure: Service Bus, multi-tenant App Services, CosmosDB, and cross-service integrations.

## Scope

- SmartFran Cloud multi-tenant App Services — franchise onboarding, per-tenant routing/config diagnostics
- Azure Service Bus — namespaces, topics, DLQ monitoring, action groups
- CosmosDB — availability, RU consumption
- Event-driven messaging and cross-service integrations across the SmartFran Cloud stack

For Azure AD Domain Services, VMs, and NSGs, use `/ope-azure`. For NSG-specific audits on `sfcgnetsec01`, use `/loyalty-azure-nsg`.

## Known Infrastructure

| Resource | Value |
|---|---|
| Subscription | SmartIT Cloud (`85c76dea-3304-4310-8656-bf21b28e4f4b`) |
| SmartFran Cloud shared services RG | `SmartFran.Cloud.PRO` (all shared App Services + Key Vault + App Configuration + Redis + SignalR + Cosmos) |
| SmartFran Cloud Key Vault | `SmartFran-Cloud-KV-Pro2` |
| SmartFran Cloud App Configuration | `SmartFran-Cloud-Settings-PRO` |
| SmartFran Cloud logging VM | `smartfran-graylog-pro` (Graylog, eastus2) |
| Service Bus namespace | `SmartFran-Cloud-ServiceBus-Grido-PRO` |
| Service Bus RG | `SmartFran.Cloud.PRO.GRIDO` |
| Service Bus action group | `Service_Bus_Group` (RG: `SmartFran.Cloud.PRO`) |

## Command Constraint

> **Read commands** (`az ... show`, `az ... list`, `az ... describe`) — safe to run, no warning needed.
> **Write/action commands** (`az ... update`, `az ... set`, `az ... delete`, `az ... restart`) — always prefix the block with `⚠️ ACTION — this command modifies Azure state.`

---

## SmartFran Cloud — Multi-tenant Franchise Architecture

Full architecture reference (shared App Services, per-tenant hostname pattern, persistent data isolation, App Insights naming convention): **`cloud/docs/infrastructure.md`** — read it before investigating a new franchise, and update it (not this file) when a new architecture fact is confirmed.

### New franchise onboarding / per-tenant 404 diagnostic workflow

Verified against source (`repo/SmartFran.Cloud/`, 2026-07-12): tenant resolution is **not** hostname-based at the API layer — it's per-request, from an Auth0 JWT `org_id` claim or a `TenantId` header, matched against a single App Configuration key called **`organizations`** (JSON array of `{name, orgId, tenantId}`). `tenantId` is a 12-char opaque hash, not the franchise name. Full detail: `cloud/docs/infrastructure.md` → "Tenant resolution mechanism".

**First, tell apart the two failure classes** — they need different fixes:
- **Static content 404** (e.g. `manifest.webmanifest`, any `wwwroot/*` asset) → front-end deployment problem for that specific client project (e.g. `Pos.Wasm`). Has nothing to do with the `organizations` config.
- **`ApiTenantException` / `TenantNotAvailable`** (backend) → the `organizations` array is missing the tenant, has a malformed id, or the caller's `org_id`/`TenantId` doesn't match anything in it.

1. **Inventory both RGs** — `az resource list -g SmartFran.Cloud.PRO.<TENANT>` (persistent data) and `-g SmartFran.Cloud.PRO` (shared services) — confirm what actually exists.
2. **Hostname/cert binding** — `az webapp config hostname list --webapp-name <app> -g SmartFran.Cloud.PRO`. If the tenant's hostname isn't `Verified`/`SniEnabled` here, stop — it's a DNS/cert issue, not app config. (This only affects which build is served — it does not affect backend tenant resolution.)
3. **`organizations` config presence** — read the actual `organizations` key value from App Configuration and check whether the tenant's `name`/`orgId`/`tenantId` triplet is present and well-formed (12 alphanumeric chars). `az appconfig kv list` often needs `--auth-mode login` explicitly.
4. **Secrets presence** — `az keyvault secret list --vault-name SmartFran-Cloud-KV-Pro2 --query "[].name" -o tsv | grep -i <tenant>`. Also check `ConnectionStrings:TenantConnection_<tenantId>` etc. exist for the resolved `tenantId`, not the franchise name.
5. **App Insights — get the actual failing request**, don't diagnose from a log-line snippet alone: query the `_new`-suffixed component's `requests`/`dependencies` tables filtered by `resultCode == '404'`. For an `SFC.<Target>` HttpClient failure (`System.Net.Http.HttpClient.Sfc.<Target>.LogicalHandler` in logs), check `dependencies` with `name contains 'Sfc.<Target>'`.
6. **Before concluding it's tenant-specific: check whether a known-good tenant hits the same error.** A 404 that looks like a WEISS-only onboarding gap can just as easily be an unrelated stale/unapplied deployment affecting the shared App Service for everyone (or everyone who hasn't refreshed a cached client) — confirmed as the actual root cause in the 2026-07-11 case. Don't build the whole remediation around a tenant-config theory until this is ruled out.

```bash
# C1: Resources in a franchise's own persistent-data RG
az resource list --resource-group SmartFran.Cloud.PRO.<TENANT> \
  --query "[].{Name:name, Type:type, Location:location}" --output table

# C2: Resources in the shared services RG
az resource list --resource-group SmartFran.Cloud.PRO \
  --query "[].{Name:name, Type:type, Location:location}" --output table

# C6: Hostname/cert binding — confirm DNS/TLS layer, per shared app
az webapp config hostname list --webapp-name <app-name> \
  --resource-group SmartFran.Cloud.PRO --output table

# C7: Read the actual `organizations` value from App Configuration
az appconfig kv show --name SmartFran-Cloud-Settings-PRO --auth-mode login \
  --key organizations --output json

# C10: Key Vault — confirm tenant DB connection secrets exist
az keyvault secret list --vault-name SmartFran-Cloud-KV-Pro2 \
  --query "[].name" --output tsv | grep -i <tenant>

# C11: App Insights — the actual failing front-end request (last 24h)
az monitor app-insights query \
  --app <AppService>_new -g SmartFran.Cloud.PRO \
  --analytics-query "requests | where timestamp > ago(24h) and resultCode == '404' | project timestamp, name, url, resultCode, cloud_RoleInstance | order by timestamp desc | take 50" \
  --output table

# C12: App Insights — dependency (backend HttpClient / SFC.* target) failures
az monitor app-insights query \
  --app <AppService>_new -g SmartFran.Cloud.PRO \
  --analytics-query "dependencies | where timestamp > ago(24h) and resultCode == '404' | project timestamp, name, target, resultCode, data, cloud_RoleInstance | order by timestamp desc | take 50" \
  --output table
```

---

## Windows App Service — console logs can never reach Graylog via this pipeline (platform limitation, not config)

All SmartFran Cloud services share one Serilog pipeline (Console JSON → stdout → `AppServiceConsoleLogs` → Event Hub → Graylog). **This cannot work for Windows App Services (Sales, Pos) at all** — `AppServiceConsoleLogs` isn't a supported/populated Diagnostic Settings category for .NET on Windows (Microsoft only supports it for JavaSE/Tomcat there). Confirmed 2026-08-11 (GITIN-1811), for **Sales-DEV** — full detail in `cloud/docs/infrastructure.md` → "Logging (GSFC-LOG-1)" and `cloud/events/20260810_sales-serilog-console-logs/`. **Linux** App Services (Business, Platform, Person, Admin, Catalog, Orders) work fine — that category *is* supported for containers; don't waste time on any of this for them.

Don't chase this as a config problem — three real gates (`stdoutLogEnabled`, `applicationLogs.fileSystem`, ANCM stdout buffering under `hostingModel="inprocess"`) were found and fixed, confirmed with real CLEF JSON visible live via Kudu Log Stream, and it *still* never reached Graylog after an hour of real traffic. The category itself doesn't ship data off Windows/.NET App Services, full stop. If asked to investigate this again for Sales/Pos (any environment), the diagnostic commands below are still useful for confirming ANCM capture works locally-in-the-file, but **do not present a `web.config`/Application-Logging fix as a solution** — it isn't one. Resolution needs a dev decision (direct GELF sink in Serilog, or the unconfirmed `AppServiceAppLogs`+`AzureMonitorTraceListener` path, or something else) — see `ops.md` in the event folder.

`Sales-PRO`/`Pos-PRO` are unverified, separate from DEV, but subject to the same platform limitation if checked — **always confirm which environment before running these commands**, the RG/subscription differ (PRO: `SmartFran.Cloud.PRO`, subscription `SmartIT Cloud` `85c76dea...`; DEV: `SmartFran.Cloud`, subscription `Smart IT - Grido` `0190fa7d...`) and mixing them up wastes a round of diagnosis on the wrong instance.

```bash
# Set both together — mixing an app name from one environment with the
# other's RG/subscription silently queries the wrong resource.
APP=SmartFran-Cloud-Sales-DEV; RG=SmartFran.Cloud; SUB=0190fa7d-4ccf-4e3d-beb1-323b5780bfc8   # DEV
# APP=SmartFran-Cloud-Sales-PRO; RG=SmartFran.Cloud.PRO; SUB=85c76dea-3304-4310-8656-bf21b28e4f4b   # PRO

# C1: Read the live web.config via Kudu VFS (AAD token — avoids exposing
# publishing-credentials passwords via list-publishing-credentials)
TOKEN=$(az account get-access-token --resource https://management.azure.com/ --query accessToken -o tsv)
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://$(echo "$APP" | tr '[:upper:]' '[:lower:]').scm.azurewebsites.net/api/vfs/site/wwwroot/web.config"

# C2: Application Logging (Filesystem) — second gate; must also not be "Off"
# for the platform to collect the stdout file even if capture is enabled
az webapp log show --name $APP --resource-group $RG --subscription $SUB -o json
```

Sources: [Azure Web app (Windows) console logs not showing in AppServiceConsoleLogs — Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/2147544/azure-web-app-(window)-asp-net-api-console-logs-ar), [AppServiceAppLogs is now available for ASP .NET apps on Windows](https://azure.github.io/AppService/2020/08/31/AzMon-AppServiceAppLogs.html).

---

## Azure Service Bus — DLQ Monitoring

**Portal:** Service Bus namespace → **Metrics** → metric `Dead-lettered messages`, split by Entity Name (per topic/queue). For prebuilt dashboards: namespace → **Insights**.

```bash
# List topics in a namespace
az servicebus topic list \
  --namespace-name SmartFran-Cloud-ServiceBus-Grido-PRO \
  --resource-group SmartFran.Cloud.PRO.GRIDO \
  --output table

# List alert rules scoped to the Service Bus namespace
az monitor metrics alert list \
  --resource-group SmartFran.Cloud.PRO.GRIDO \
  --output table \
  --query "[].{Name:name, Enabled:enabled, Severity:severity, Criteria:criteria}"

# Show action group receivers
az monitor action-group show \
  --name Service_Bus_Group \
  --resource-group SmartFran.Cloud.PRO \
  --output json
```

**DLQ alert rule baseline:** metric `Dead-lettered messages`, threshold `> 0`, aggregation `Total`, evaluation window ≥ 5 min. Narrower windows risk hitting the action group's email rate cap during bursts — see `/ope-azure` → "Azure Monitor — Action Group Email Warnings" for the limits.

---

## Constraints

- Default to read-only commands.
- Always warn with `⚠️ ACTION` before any command that modifies state.
- Never generate commands that delete resources or force-restart services unless explicitly requested.
- Output commands as copy-paste blocks. User runs them and pastes results back.
