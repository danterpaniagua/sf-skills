# Skill: Cloud POS Invalid Sale Diagnosis

Diagnostic workflow for SmartFran Cloud POS sales rejected as invalid ("La venta es invalida" / `SaleIsInvalid`, code 12000) when a discount, promotion, or combo is involved. Built from a real case (WEISS, `20260802_promocion-invalida-weiss-franui`): a mispriced discount item, not a code defect or promotion eligibility rule, was the cause — but a naive mockup of the fix initially made a wrong prediction, which is itself the most useful lesson here (see "Two totals" below).

## Scope

- POS checkout rejections tied to combos, promotions, or discount/financial-modifier line items.
- Distinguishing a genuine code/validation defect from a Catalog/Business data-configuration error.
- Not for infra-level issues (App Service, Service Bus, CosmosDB availability) — use `/cloud-azure` for those.

## The error itself is generic — don't over-read it

`SaleIsInvalid` (12000, "La venta es invalida") is thrown by `SaleService.CreateAsync` (`Source/Services/Sales/.../SaleService.cs`) whenever `SaleCreateCmdValidator` (FluentValidation) fails **any** rule. All FluentValidation failures collapse into this one generic exception/message — the POS dialog title ("¡Atención! No se cumple una validación.", `MainApp.razor`) is a hardcoded wrapper, not a promotion-specific message. **Do not assume the text tells you which rule failed.**

No combo/discount mutual-exclusion or eligibility re-validation exists server-side. `PromotionGroup`/`PromotionDetail` config is POS-UI-only, never re-checked at `SaleCreateCmd` time. In practice, a sale involving a discount/promotion almost always fails because **payments collected don't match a recomputed line-item total** — but there are **two different, non-interchangeable "total" calculations** in play, and confusing them is the single easiest way to get this diagnosis wrong.

## Two totals — do not conflate them

| | Formula | Used for |
|---|---|---|
| **Display/charge total** | `GetTotal()` (`Sale.razor.cs`): `Σ c.Total` over non-body lines, discount folded in via `CalcAmountWithOutIva` | What the cashier sees and what the customer is actually asked to pay |
| **Validation total** | `SaleCreateCmdValidator` / client-side `ValidateAmount`: `Σ (Quantity × UnitPrice)` over `Type != 1 \|\| (Type == 1 && TypeDetail == 6/*Extra*/)` | The `payments >= total` check that actually accepts/rejects the sale |

These use different fields (`.Total` vs `UnitPrice × Quantity`) and can legitimately disagree, especially once a combo/promo header+body decomposition is involved — combo body lines get `UnitPrice` **recomputed** as `Total / Quantity` and re-rounded during Details construction (`Sale.razor.cs`'s `TypeDetail` switch), while a plain product line's `UnitPrice` is untouched.

**Consequence for any mockup/simulation you build:** modeling only the display-total formula and assuming it equals what the validator checks will produce wrong predictions. In the WEISS case, a mockup that computed the discount residual via the display formula correctly reproduced the exact reported amounts ($0.14 / $0.17 residual, bit-for-bit) but **wrongly predicted rejection for a case that actually passed** — because that case's real acceptance hinged on the validator's separate `UnitPrice × Quantity` recomputation happening to land on a different number than the display total. Don't declare a plain/combo pass-fail divergence "confirmed" from a mockup unless you've modeled the validation-total formula specifically, with real per-line `UnitPrice`/`Quantity` values (not `.Total`) — which usually means you need the actual `SaleCreateCmd` payload (App Insights), not just Catalog/Business DB prices.

## Diagnostic order — cheapest/most-likely first

1. **Check the discount/financial-modifier item's actual configured price before anything else.** If a "100% off" or "free item" discount is involved, query its real price — don't trust the item's name. A discount item literally named "...100%" was found priced at `99.99`, not `100.00` (Query 2 below).
2. **Build a small mockup replaying the real formula with real numbers before calling anything confirmed.** A percentage-based discount bug is easy to verify deterministically: if all relevant prices are clean whole-dollar amounts (check — they usually are), the shortfall is exactly `subtotal × (100 - discount%) / 100`, no floating-point ambiguity. This turns "plausible" into "exact, reproducible to the cent." But see "Two totals" above before trusting a mockup's pass/fail prediction, not just its residual-amount prediction.
3. **Test the same combination on a plain-item line vs. a combo-structured line for the same product**, if both variants exist (e.g. "Franui" vs "Franui c/Combo"). A residual that's invisible on combo lines but hard-fails on plain lines is diagnostic of the two-totals divergence above — but confirming *why*, precisely, needs the real payload (see "Getting real per-line data" below), not just DB prices.
4. **Payment method is very likely a red herring — test it, don't assume it.** `ValidateAmount`'s client-side mismatch handling only self-heals a payment line literally named "Efectivo" (cash). In the confirmed case, cash failed identically to a bank transfer — the mismatch was a genuine amount owed, not an unhealed rounding gap. Test both before building a theory around payment method.
5. **If the item is promotion-linked, check the `Promotions` schedule/activation window** (`ValidSinceDate`/`ValidToDate` vs `ActivatedDate`/`DeactivatedDate`) — but only as a parallel check. An expired-but-activated / valid-but-never-activated promotion pair is a real and separate class of bug worth flagging, but confirm it's actually in the failing sale's path before attributing the rejection to it.
6. **The failed sale itself is not in SQL — and getting its real per-line data is harder than it looks.** `Sale`/`SaleDetail` are CosmosDB documents (Sales service), and since `SaleCreateCmdValidator` throws *before* `_saleRepository.AddAsync`, a **rejected** sale never persists, not even in Cosmos. See "Getting real per-line data" below before spending time on Graylog/App Insights for the exception content specifically — it's a weaker path than it looks, and there's a better one.

## Getting real per-line data (`UnitPrice`/`Quantity`/`Type`/`TypeDetail`) — don't start with logs for the exception content

**Do not assume Graylog or App Insights has the actual exception text or request payload.** In the WEISS case, `SaleService.CreateAsync`'s catch block only does `_log.LogError(null, e, e.Message)` — confirmed by checking Sales' `AppServiceAppLogs` in Graylog directly: every entry is ASP.NET Core framework pipeline tracing (`Start/End processing HTTP request`, `Executed action method`, etc. — same finding as `20260728_logging-verbosity-ef-core-cors`), plus a generic Kestrel `"An unhandled exception was thrown by the application."` line with **`message: null`** in its `properties` — the actual exception text/stack trace is not captured in this log category at all. Don't burn time iterating search phrases against it once you've confirmed this pattern once for a given app/tenant.

**Two traps when reading these logs:**
- A log line saying `"...Validation state: Valid"` (e.g. `"Executing action method ...CreateAsync ... - Validation state: Valid"`) refers to ASP.NET Core **model-binding**/DataAnnotations validation of the DTO shape — it has nothing to do with `SaleCreateCmdValidator`'s FluentValidation business rules, which run later inside the action method. Seeing this line does **not** mean the `payments >= total` check passed.
- A generic `"An unhandled exception was thrown by the application"` entry near your test's timeframe is not necessarily *your* exception — cross-check the surrounding log lines (same request, by timestamp) for what code path was actually executing. In the WEISS investigation, two such entries turned out to be an unrelated `AddFriendlyCode` failure (a different, transient issue), not the discount-related `SaleIsInvalid` rejection — confirmed by seeing the preceding `"Generando FriendlyCode para Sale..."` line, which only executes *after* the FluentValidation check already passed.

**The better path: query CosmosDB directly for a *passing* sale.** A sale that succeeds *does* persist — so instead of chasing the failing payload, pull the real `Details[]` from an accepted test sale on the same POS terminal (e.g. the combo+discount case that passed). This gives ground-truth `Type`/`TypeDetail`/`UnitPrice`/`Quantity`/`Total` values with no logging gaps involved.

**Critical caveat that cost significant time in the WEISS investigation: `SMARTFRAN-CLOUD-SALES-PRO` is a shared App Service serving every tenant (GRIDO, WEISS, and others), and its Graylog logs carry no tenant-identifying field at all** (`resourceId`/`name`/`category` are always the same shared-app values regardless of which tenant made the request). So a `FranchiseeCode`/`FranchiseCode`/`PosCode` triplet pulled from a log line — even from a confirmed-200 `Sale/Create` request at the right timestamp — **cannot be assumed to belong to the tenant you're investigating.** In the WEISS case, this produced two consecutive zero-result Cosmos queries against genuinely-persisted sales, because the codes almost certainly belonged to a *different* tenant's interleaved traffic in the same shared log stream. Confirming container/database/serialization were all correct (see below) is what narrowed it down to this explanation.

**More reliable ways to get the right tenant's partition-key values, in order of preference:**
1. Ask whoever is running the POS test to grab the exact `FranchiseeCode`/`FranchiseCode`/`PosCode` from the browser's network tab (the `SaleCreateCmd` request body) during a live test — unambiguous, no attribution risk. **In practice this is often the only reliable option** — see the negative result below before spending time on option 2.
2. **Tried and ruled out in the WEISS investigation — do not repeat without new information:** `Business_<TENANT>` SQL does have a franchise/franchisee/sale-point schema (`Franchise`, `Franchisee`, `SalePoint` — all **singular** table names, not the EF-pluralized default; `RelatedCode` holds opaque code values keyed by `FranchiseId`/`FranchiseeId`/`SalePointId`, joined to `RelationCode` for what system each code is for). But for WEISS, every `RelatedCode` row found was Uruguay fiscal/electronic-invoicing data (`FE-PVTA`, `FEUY-IDCOMERCIO`, `FEUY-APIKEY`, etc.), and the `SalePoint`-level `RelatedCode` lookup returned **zero rows** — the opaque Sales/Cosmos routing codes are not stored anywhere reachable in this SQL schema. They may live in the `Person`/Company cloud identity system instead (`Franchise.PersonCloudId`/`CompanyCloudId` fields point there, not yet explored) — worth trying only if that identity system's schema becomes accessible.
3. Only as a last resort, the Graylog-log-line technique below — and if you use it, verify independently (e.g. by checking whether the resulting Cosmos query returns *any* row at all) before trusting the values, rather than assuming a plausible-looking match is the right tenant.

**If searching Franchise names in `Business_<TENANT>`, expect multiple candidates and pick carefully** — the WEISS lookup returned test franchises (`Test`, `SMARTCLOUD_TIENDA_TEST_WEISS...`) and a *different physical location* (`Weiss Ensenada`) alongside the actual target (`Weiss Punta del Este`). Match on the specific location name, not just a substring of the brand name.

If you do use the Graylog approach: `SaleService.AddFriendlyCode`'s `_log.LogInformation($"Generando FriendlyCode para Sale. FranchiseeCode: {...}, FranchiseCode: {...}, PosCode: {...}, ShiftNumber: {...}")` fires on every sale attempt, success or failure — but per the caveat above, treat whatever it returns as unverified until a Cosmos query confirms it.
   ```bash
   curl -s "http://127.0.0.1:9200/sales__*/_search" -H 'Content-Type: application/json' -d '{
     "size": 5,
     "query": { "bool": { "must": [
       { "term": { "name": "SMARTFRAN-CLOUD-SALES-PRO" } },
       { "match_phrase": { "message": "Generando FriendlyCode" } }
     ]}},
     "sort": [{ "time": "desc" }]
   }'
   ```

Before assuming the partition-key values are wrong when a query comes back empty, rule out the cheaper explanations first: confirm the container name from source (`grep -rn "GetContainer(" .../CosmosSDK/*Repository.cs` — for Sales it's `cosmosConnection.GetContainer("Sales")`), and confirm the entity has no `[JsonProperty]` override and the Cosmos client options don't set a camelCase serialization policy (both true for `Sale.cs`/`SaleContext.cs` at the time of this investigation) — only once those are ruled out does tenant misattribution become the leading explanation for a confirmed-200-but-zero-rows result.

4. **Confirm the Cosmos account, database, and container** for the tenant (check `cloud/docs/infrastructure.md` → "Core Shared Services" for the account name *first* — it does not follow the plain `smartfran-cloud-cosmos-<tenantId>` pattern, don't guess it):
   ```bash
   az cosmosdb sql database list --account-name <account> --resource-group SmartFran.Cloud.PRO.<TENANT> -o table
   az cosmosdb sql container show --account-name <account> --database-name Sales-<TENANT> \
     --name Sales --resource-group SmartFran.Cloud.PRO.<TENANT> --query "resource.partitionKey"
   ```
5. **Query the `Sales` container in the Azure Portal's Data Explorer** (there is no `az` CLI command for ad-hoc SQL API document queries — `az cosmosdb sql *` only manages account/database/container metadata):
   ```sql
   SELECT c.id, c._ts, c.ShiftNumber, c.FriendlyCode, c.Details, c.MoneyMovementData
   FROM c
   WHERE c.FranchiseeCode = "<value>" AND c.FranchiseCode = "<value>" AND c.PosCode = "<value>"
   ORDER BY c._ts DESC
   ```
   See "Databases and confirmed table/column names" below for the full container list, and `cloud/docs/infrastructure.md` for the canonical, versioned copy.

## Tooling note: Graylog/OpenSearch field names — don't assume, check a sample doc first

Confirmed field names differ from what you'd guess: the event's real timestamp is **`time`** (ISO8601, e.g. `"2026-08-02T23:19:58Z"`) — **not** `timestamp`, which is Graylog's own *receive* time in a custom, awkward-to-query format (`uuuu-MM-dd HH:mm:ss.SSS`) that will throw a `parse_exception` if you feed it a hand-typed literal instead of a relative expression like `now-2d`. Always pull one sample doc first (`{"query":{"match_all":{}}, "sort":[{"time":{"order":"desc"}}], "size":1}`) to confirm field names/format before building a filtered query — this cost several failed queries in the WEISS investigation.

## Databases and confirmed table/column names

Two separate SQL Server databases per tenant (elastic pool `t102-smartfran-cloud-<tenant>`) — getting the wrong database is the single most common mistake here (see tooling note below):

**`SmartFran.Cloud.Business_<TENANT>`** — promotion/combo definitions:

| Table | Key columns |
|---|---|
| `Promotions` | `Name`, `PromotionType` (0=Promotion,1=Combo), `ValidSinceDate`/`ValidToDate`, `ValidSinceMinute`/`ValidToMinute`, day-of-week bools, `MandatoryForAll`, `ActivatedDate`/`DeactivatedDate` |
| `PromotionGroups` | `PromotionId` (FK), `Amount`, `Type` (string: FixedPrice/FixedDiscount/PercentDiscount/PriceList), `HasAdditionals`, `MultipleSelection` |
| `PromotionDetails` | `PromotionGroupId` (FK), `ArticleId` — eligible articles per group |
| `PromotionApplies` | `Include`, `FranchiseId`, `FranchiseeId`, `PriceListId`, etc. |
| `PromotionGroupsApplies` | note the name — "GroupsApplies", not "GroupApplies". **`PromotionValue`** — this is a *third*, independent pricing source (see below) |
| `Oversales` / `OversaleComboPromotionItems` | Upsell-prompt config — usually a distractor, not the cause (governs cashier-facing upsell offers, not manually-added combo contents) |

**`SmartFran.Cloud.Catalog_<TENANT>`** — item/price/financial-modifier definitions:

| Table | Key columns |
|---|---|
| `Items` | `Name`, `GroupId`, `ForSale` |
| `Groups` | `FinancialModify` (0=NoApply, 1=Discount, 2=Surcharge, 3=Delivery) — this is how a "discount item" is distinguished from a normal product |
| `PriceLists` / `PriceDetails` | `PriceDetails.ItemId`/`PriceListId`, **`PublishedPrice`/`NewPrice`** (not `Price` — that column doesn't exist), `Enabled` |

**Important — an article can have up to three different "prices" depending on how it's sold, don't assume `PriceDetails.PublishedPrice` is the only one:**
1. `PriceDetails.PublishedPrice`/`NewPrice` (Catalog) — the article's standalone retail price.
2. `PromotionGroupsApplies.PromotionValue` (Business), where `PromotionGroups.Type = 'FixedPrice'` — the price of that same article **when it's an eligible component inside a specific combo/promotion**, scoped to a `PriceListId`. In the WEISS case, one article had a standalone price of `9900` in a given price list, but a `PromotionValue` of `250` for the *same price list* when sold as a combo add-on — a ~40x difference. Confusing these two is an easy way to build a mockup with wildly wrong numbers.
3. Article eligibility (`PromotionDetails.ArticleId`) can appear across dozens of different `Promotions` rows simultaneously (one article can be an add-on option for many unrelated combos) — don't assume a name search on `Promotions` will find every place an article is priced; searching `PromotionDetails.ArticleId` directly is more reliable once you have the article's `Id`.

```sql
-- Query 1: locate a promotion/combo by name, and check its activation vs. date-window state
SELECT Id, Name, Description, PromotionType, MandatoryForAll,
       ValidSinceDate, ValidToDate, ActivatedDate, DeactivatedDate
FROM Promotions
WHERE Name LIKE '%<term>%' OR Description LIKE '%<term>%';

-- Query 2: locate a discount/financial-modifier item and its REAL configured standalone price
-- (run against Catalog_<TENANT> — confirm with `SELECT DB_NAME()` first if unsure)
SELECT i.Id, i.Name, i.GroupId, g.FinancialModify, g.Name AS GroupName
FROM Items i
LEFT JOIN Groups g ON g.Id = i.GroupId
WHERE i.Name LIKE '%<term>%';

SELECT pd.ItemId, pd.PriceListId, pd.PublishedPrice, pd.NewPrice, pd.Enabled
FROM PriceDetails pd
JOIN Items i ON i.Id = pd.ItemId
WHERE i.Name LIKE '%<term>%';

-- Query 3 (run against Business_<TENANT>): find every combo/promo that prices this same
-- article as a component, and at what value per price list — use the article's Id from Query 2
SELECT p.Id, p.Name, p.PromotionType, p.ActivatedDate, p.DeactivatedDate,
       g.Id AS GroupId, g.Amount, g.Type AS GroupType,
       d.ArticleId, ga.PromotionValue, a.PriceListId
FROM PromotionDetails d
JOIN PromotionGroups g ON g.Id = d.PromotionGroupId
JOIN Promotions p ON p.Id = g.PromotionId
LEFT JOIN PromotionGroupsApplies ga ON ga.PromotionGroupId = g.Id
LEFT JOIN PromotionApplies a ON a.Id = ga.PromotionApplyId
WHERE d.ArticleId = <item id>;
```

**CosmosDB (Sales service, SQL API)** — where actual sale transactions live, as opposed to promotion/item *definitions* above:

| Database | Container(s) | Notes |
|---|---|---|
| `Sales-<TENANT>` | `Sales` — partition key `[FranchiseeCode, FranchiseCode, PosCode]` (the container this skill actually queries) | Full container list for `Sales-<TENANT>` is in `cloud/docs/infrastructure.md` (authoritative). Only **passing** sales appear in `Sales` — see "Getting real per-line data" above for rejected sales. |

## Tooling note: `USE` does not switch databases in Azure SQL

These tenant databases are **Azure SQL Database**, not on-prem/IaaS SQL Server — each database is a fully separate connection scope, and a `USE [OtherDatabase];` statement inside a query is silently a no-op (it does not error, it just doesn't do anything). If a query fails with "Invalid object name" for a table you know exists, the first thing to check is which database the query tool is actually connected to (`SELECT DB_NAME();`), and switch it via the tool's **connection picker**, not via SQL.

## Worked example

Full investigation, including every dead end, the eventual mockup validation, and where that mockup's prediction was wrong: `cloud/events/20260802_promocion-invalida-weiss-franui/` (`_investigation.md` for the narrative, `_scripts.py` for the runnable mockup). Confirmed exactly: a discount item named "...100%" was priced at `99.99`, and the residual this leaves on a clean whole-dollar subtotal is deterministically `subtotal × 0.0001` — reproduced the real reported amounts ($0.14 / $0.17) to the cent. **Not fully confirmed**: the precise reason a combo-structured line passes while a plain-item line fails with the same discount — the leading explanation (validation-total vs. display-total divergence) is well-supported by source but wasn't closed with real payload data.

## Constraints

- Read-only by default. Any `UPDATE`/`DELETE` against Catalog/Business data is a production data write — confirm with whoever owns pricing/promotions for that tenant before proposing it as final, and always show a `SELECT` preview of affected rows first.
- Output SQL as copy-paste blocks. The user runs it and pastes results back — never execute directly.
- All content written to `cloud/events/` follows that project's Spanish/English split — see `cloud/CLAUDE.md`.
