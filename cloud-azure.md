# Skill: Cloud Azure Operations

Investigation and remediation support for the SmartFran Cloud platform on Azure: Service Bus, multi-tenant App Services, CosmosDB and cross-service integrations.

## Scope

- Multi-tenant App Services: franchise onboarding, per-tenant routing/config diagnostics
- Service Bus: namespaces, topics, DLQ monitoring, action groups
- CosmosDB: availability, firewall, RU consumption
- Event-driven messaging and cross-service integrations

For Azure AD Domain Services, VMs and NSGs use `/ope-azure`; for NSG audits on `sfcgnetsec01`, `/loyalty-azure-nsg`; for the Auth0 permission audit, `/cloud-auth0`.

## Known infrastructure

Subscription **SmartIT Cloud** (`85c76dea-3304-4310-8656-bf21b28e4f4b`); shared services RG `SmartFran.Cloud.PRO` (App Services, Key Vault `SmartFran-Cloud-KV-Pro2`, App Configuration `SmartFran-Cloud-Settings-PRO`, Redis, SignalR, Cosmos, logging VM `smartfran-graylog-pro`, eastus2); Service Bus namespace `SmartFran-Cloud-ServiceBus-Grido-PRO` in RG `SmartFran.Cloud.PRO.GRIDO`, action group `Service_Bus_Group` (RG `SmartFran.Cloud.PRO`). Everything else (resource groups, App Services and plans, Function Apps, tenant resolution, Cosmos account names, Auth0, Graylog VM and index sets): **`cloud/docs/infrastructure.md`**. Read it before investigating, and update it (not this file) when a new architecture fact is confirmed.

## Command constraint

- **Read commands** (`az ... show`, `list`, `describe`): safe, no warning.
- **Write/action commands** (`az ... update`, `set`, `delete`, `restart`): prefix the block with `⚠️ ACTION — this command modifies Azure state.`

---

## New franchise onboarding / per-tenant 404 diagnostic workflow

Tenant resolution is **not** hostname-based at the API layer: it is per request, from the Auth0 JWT `org_id` claim or the `TenantId` header, matched against one App Configuration key, **`organizations`** (JSON array of `{name, orgId, tenantId}`; `tenantId` is a 12-char opaque hash, not the franchise name). Detail: `cloud/docs/infrastructure.md` → "Tenant resolution mechanism".

**First tell apart the two failure classes:**
- **Static-content 404** (`manifest.webmanifest`, any `wwwroot/*` asset): a front-end deployment problem of that client project (e.g. `Pos.Wasm`), unrelated to `organizations`.
- **`ApiTenantException` / `TenantNotAvailable`** (backend): the `organizations` array lacks the tenant, has a malformed id, or the caller's `org_id`/`TenantId` matches nothing.

1. **Inventory both RGs**: `az resource list -g SmartFran.Cloud.PRO.<TENANT>` (persistent data) and `-g SmartFran.Cloud.PRO` (shared services).
2. **Hostname/cert binding**: `az webapp config hostname list`. If the hostname is not `Verified`/`SniEnabled`, stop: a DNS/cert issue, not app config (it only affects which build is served, not backend tenant resolution).
3. **`organizations` presence**: read the real key value and check the tenant's `name`/`orgId`/`tenantId` triplet is present and well formed (12 alphanumeric chars). `az appconfig kv list` often needs `--auth-mode login`.
4. **Secrets presence**: `az keyvault secret list ... | grep -i <tenant>`, and that `ConnectionStrings:TenantConnection_<tenantId>` etc. exist for the resolved `tenantId` (not the franchise name).
5. **App Insights: get the actual failing request**, not a log-line snippet: query the `_new`-suffixed component's `requests`/`dependencies` filtered by `resultCode == '404'`. For an `SFC.<Target>` HttpClient failure (`System.Net.Http.HttpClient.Sfc.<Target>.LogicalHandler` in logs) check `dependencies` with `name contains 'Sfc.<Target>'`.
6. **Before calling it tenant-specific, check whether a known-good tenant hits the same error.** A WEISS-looking 404 was an unapplied/stale deployment of the shared App Service affecting everyone (the 2026-07-11 root cause). Do not build the remediation around a tenant-config theory until this is ruled out.

```bash
# C1: resources in a franchise's persistent-data RG
az resource list --resource-group SmartFran.Cloud.PRO.<TENANT> \
  --query "[].{Name:name, Type:type, Location:location}" --output table

# C2: resources in the shared services RG
az resource list --resource-group SmartFran.Cloud.PRO \
  --query "[].{Name:name, Type:type, Location:location}" --output table

# C6: hostname/cert binding per shared app
az webapp config hostname list --webapp-name <app-name> \
  --resource-group SmartFran.Cloud.PRO --output table

# C7: the actual `organizations` value
az appconfig kv show --name SmartFran-Cloud-Settings-PRO --auth-mode login \
  --key organizations --output json

# C10: Key Vault: tenant DB connection secrets exist
az keyvault secret list --vault-name SmartFran-Cloud-KV-Pro2 \
  --query "[].name" --output tsv | grep -i <tenant>

# C11: App Insights: the failing front-end request (last 24h)
az monitor app-insights query \
  --app <AppService>_new -g SmartFran.Cloud.PRO \
  --analytics-query "requests | where timestamp > ago(24h) and resultCode == '404' | project timestamp, name, url, resultCode, cloud_RoleInstance | order by timestamp desc | take 50" \
  --output table

# C12: App Insights: dependency (backend HttpClient / SFC.* target) failures
az monitor app-insights query \
  --app <AppService>_new -g SmartFran.Cloud.PRO \
  --analytics-query "dependencies | where timestamp > ago(24h) and resultCode == '404' | project timestamp, name, target, resultCode, data, cloud_RoleInstance | order by timestamp desc | take 50" \
  --output table
```

## Other failure classes already solved (details in `cloud/docs/infrastructure.md`)

- **Cosmos `403 Forbidden` / `CosmosException`** on a tenant account: read the account firewall (`az cosmosdb show`: `ipRules`, `networkAclBypass`, `virtualNetworkRules`) against the apps' `possibleOutboundIpAddresses` (GITIN-2083).
- **Graylog `502` / ingestion stopped:** VM storage and the OpenSearch mount (`fstab`), and the flood-stage `read_only_allow_delete` block (GITIN-2128, "Graylog VM operations").
- **Sales-PRO crashes (`OutOfMemoryException`):** check `use32BitWorkerProcess` after every deploy (`az webapp config show`).
- **POS 403 after a release:** scope enforcement on `GET /api/Tenant/{id}` and the PRO Auth0 tenant (GITIN-2134; `/cloud-auth0`).

## Onboarding App Services to Graylog — traps confirmed (GITIN-1794 DEV, GITIN-1834 PRO)

Terraform and pipeline code live in the separate `cloud-graylog` repo (`~/Documentos/git/cloud-graylog/`; its `CLAUDE.md` holds the per-app table). Pattern: Diagnostic Settings → Event Hub `app-logs` → Logstash → Graylog. Events: `cloud/events/20260810_dev-environment-onboarding/`, `20260812_prod-full-onboarding/`.

- **A Diagnostic Setting is a full-replace PUT keyed by `name`.** Applying a new one named like an existing out-of-band Log Analytics setting (`SmartFran.Cloud.PRO.{App}_DiagnosticSettings`) silently overwrote it on 4 of 5 apps; `terraform plan` cannot warn (the old object is not in state). Run `az monitor diagnostic-settings list` per app first; keep both destinations and all 9 log categories in the one object. One survivor (Person) had a leading space in its name.
- **`workspaceId` is not validated on read.** An existing setting pointed to a subscription that does not resolve (`d6ab1add…`, `SubscriptionNotFound`) and the apply failed with 404; the real workspace is in `85c76dea…`. Confirm with `az monitor log-analytics workspace show --subscription 85c76dea…`.
- **DEV is RG `SmartFran.Cloud` in `Smart IT - Grido` (`0190fa7d…`), not `SmartFran.Cloud.TEST`** (`SmartIT Cloud`, 2 stopped apps): 33 apps over DEV/DEV2/STG/TEST/POC; scope was the 8 `*-DEV` apps. Terraform needs a second aliased provider for that subscription.
- **New stream, in this order:** create it with an exact-match rule (`field: name, type: 1`) and `remove_matches_from_default_stream: false`; check with a stream-filtered search that it holds only that app; only then set the flag to `true` (it affects new messages only: a 5-minute check still shows old ones, use a 60-second window). API quirks: `POST /streams` needs `{"entity": {...}}`; `POST /system/indices/index_sets` rejects `can_be_default` and requires non-null `data_tiering`.
- **`POST /api/authz/shares/entities/{grn}` is not a safe read.** It replaces the grant set: an empty `{}` cleared a share. Always send the full desired `selected_grantee_capabilities` (Everyone = `grn::::builtin-team:everyone`, Viewer = `view`); `GET` returns 405.
- **Volume after onboarding:** compare by category before blaming the pipeline: Catalog was 86.7 % of messages, `AppServiceConsoleLogs` 44,904 vs `AppServiceHTTPLogs` 280 in 15 minutes (app-level logging verbosity, not traffic).

---

## Windows App Service — console logs cannot reach Graylog through this pipeline (platform limitation, not config)

All services share one Serilog pipeline (Console JSON → stdout → `AppServiceConsoleLogs` → Event Hub → Graylog). **It cannot work for Windows App Services (Sales, Pos):** `AppServiceConsoleLogs` is not supported for .NET on Windows (Microsoft supports it there only for JavaSE/Tomcat). Confirmed on **Sales-DEV** (GITIN-1811); **Linux** App Services (Business, Platform, Person, Admin, Catalog, Orders) work, so do not spend time on this for them. Detail: `cloud/docs/infrastructure.md` → "Logging (GSFC-LOG-1)", `cloud/events/20260810_sales-serilog-console-logs/`.

Do not chase it as config: three real gates (`stdoutLogEnabled`, `applicationLogs.fileSystem`, ANCM stdout buffering under `hostingModel="inprocess"`) were fixed and the CLEF JSON became visible in Kudu Log Stream, and it still never reached Graylog after an hour of real traffic. Use the commands below only to confirm local capture, and **never present a `web.config`/Application-Logging change as the solution**. Resolution needs a dev decision (a direct Serilog→GELF/Event-Hub sink, as Sales now has, or the unconfirmed `AppServiceAppLogs` + `AzureMonitorTraceListener` path).

`Sales-PRO`/`Pos-PRO` are unverified but subject to the same limitation. **Confirm the environment first:** PRO is RG `SmartFran.Cloud.PRO` in `SmartIT Cloud` (`85c76dea…`); DEV is RG `SmartFran.Cloud` in `Smart IT - Grido` (`0190fa7d…`). Mixing them wastes a round on the wrong instance.

```bash
# Set all three together; an app name from one environment with the other's RG/subscription silently queries the wrong resource.
APP=SmartFran-Cloud-Sales-DEV; RG=SmartFran.Cloud; SUB=0190fa7d-4ccf-4e3d-beb1-323b5780bfc8   # DEV
# APP=SmartFran-Cloud-Sales-PRO; RG=SmartFran.Cloud.PRO; SUB=85c76dea-3304-4310-8656-bf21b28e4f4b   # PRO

# C1: live web.config via Kudu VFS with an AAD token (avoids publishing-credential passwords)
TOKEN=$(az account get-access-token --resource https://management.azure.com/ --query accessToken -o tsv)
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://$(echo "$APP" | tr '[:upper:]' '[:lower:]').scm.azurewebsites.net/api/vfs/site/wwwroot/web.config"

# C2: Application Logging (Filesystem); must not be "Off" for the platform to collect the stdout file
az webapp log show --name $APP --resource-group $RG --subscription $SUB -o json
```

Sources: Microsoft Q&A "Azure Web app (Windows) console logs not showing in AppServiceConsoleLogs"; Azure App Service blog "AppServiceAppLogs is now available for ASP.NET apps on Windows".

---

## Service Bus — DLQ monitoring

**Portal:** namespace → **Metrics** → `Dead-lettered messages`, split by Entity Name; prebuilt dashboards under the namespace's **Insights**.

```bash
# topics in the namespace
az servicebus topic list \
  --namespace-name SmartFran-Cloud-ServiceBus-Grido-PRO \
  --resource-group SmartFran.Cloud.PRO.GRIDO --output table

# alert rules scoped to the namespace
az monitor metrics alert list \
  --resource-group SmartFran.Cloud.PRO.GRIDO --output table \
  --query "[].{Name:name, Enabled:enabled, Severity:severity, Criteria:criteria}"

# action group receivers
az monitor action-group show \
  --name Service_Bus_Group --resource-group SmartFran.Cloud.PRO --output json
```

**DLQ alert baseline:** metric `Dead-lettered messages`, threshold `> 0`, aggregation `Total`, window ≥ 5 min; narrower windows risk the action group's email rate cap during bursts (`/ope-azure` → "Azure Monitor — Action Group Email Warnings").

---

## Constraints

- Default to read-only commands; warn with `⚠️ ACTION` before any command that modifies state.
- Never generate commands that delete resources or force-restart services unless explicitly requested.
- Output commands as copy-paste blocks; the user runs them and pastes results back.
