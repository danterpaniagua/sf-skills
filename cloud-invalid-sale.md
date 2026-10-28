# Skill: Cloud POS Invalid Sale Diagnosis

Diagnostic workflow for SmartFran Cloud POS sales rejected as invalid ("La venta es invalida" / `SaleIsInvalid`, code 12000) when a discount, promotion or combo is involved. Built from a real case (WEISS, `20260802_promocion-invalida-weiss-franui`): a mispriced discount item, not a code defect or an eligibility rule, was the cause; a naive mockup of the fix first made a wrong prediction, the most useful lesson here (see "Two totals").

## Scope

- POS checkout rejections tied to combos, promotions or discount/financial-modifier line items.
- Combo/promotion *display* defects (wrong number of options, wrong articles) with no rejection: same `Promotions`/`PromotionGroups`/`PromotionDetails` data and POS combo-builder code path (confirmed in `20260816_clasica-dijon-invalid-sale-weiss`).
- Separating a genuine code/validation defect from a Catalog/Business data-configuration error.
- Not for infrastructure (App Service, Service Bus, CosmosDB availability): use `/cloud-azure`.
- **Not for third-party marketplace/delivery-platform catalogs** (PedidosYa, Rappi, ...): see "Marketplace catalogs" below first.

## The error is generic

`SaleIsInvalid` (12000) is thrown by `SaleService.CreateAsync` (`Source/Services/Sales/.../SaleService.cs`) whenever `SaleCreateCmdValidator` (FluentValidation) fails **any** rule; all failures collapse into that one message. The POS dialog title ("¡Atención! No se cumple una validación.", `MainApp.razor`) is a hardcoded wrapper: the text does not say which rule failed.

No combo/discount mutual-exclusion or eligibility re-validation exists server-side; `PromotionGroup`/`PromotionDetail` config is POS-UI-only and never re-checked at `SaleCreateCmd` time. In practice a discount/promotion sale fails because **the payments collected do not match a recomputed line-item total**, and two non-interchangeable "totals" exist.

## Two totals — do not conflate them

| | Formula | Used for |
|---|---|---|
| **Display/charge total** | `GetTotal()` (`Sale.razor.cs`): `Σ c.Total` over non-body lines, discount folded in via `CalcAmountWithOutIva` | What the cashier sees and the customer pays |
| **Validation total** | `SaleCreateCmdValidator` / client-side `ValidateAmount`: `Σ (Quantity × UnitPrice)` over `Type != 1 \|\| (Type == 1 && TypeDetail == 6/*Extra*/)` | The `payments >= total` check that accepts or rejects the sale |

`SaleDetail.Type`: 0 = normal item, 1 = promo/combo **body** line, 2 = promo/combo **header** line. `TypeDetail` (`ItemSaleType`): Sale=0, Promotion=1, Exchange=2, Oversale=3, Coupon=4, Combo=5, Extra=6, DescuentoRecargo=7, DescuentoVoucher=8. Before submit the POS builds `Details` with a `TypeDetail` switch (`Sale.razor.cs` ~5091: `Discount = PriceList - UnitPrice`, `Total` per line) and, when `totalVenta != totalCabecerasEIndividuales`, runs `ProrrateoDetallesVenta()` (~5254), which redistributes combo header/body amounts through special cases with a proportional-split fallback. Client-side `ValidateAmount` (~2127) repeats the validator formula and never reads the per-line `Discount` field.

They use different fields and can disagree, especially for a combo/promo header+body decomposition: combo body lines get `UnitPrice` **recomputed** as `Total / Quantity` and re-rounded during Details construction (`Sale.razor.cs`, `TypeDetail` switch); a plain product line's `UnitPrice` is untouched.

**Consequence for any mockup:** modeling only the display total and assuming it equals what the validator checks gives wrong predictions. In the WEISS case a mockup using the display formula reproduced the reported residuals ($0.14 / $0.17) bit for bit but **wrongly predicted rejection for a case that actually passed**, because that case's acceptance hinged on the validator's separate `UnitPrice × Quantity` recomputation. Do not call a plain/combo pass-fail divergence "confirmed" from a mockup without modeling the validation total with real per-line `UnitPrice`/`Quantity` (not `.Total`), which usually needs the real `SaleCreateCmd` payload, not just Catalog/Business prices.

## Diagnostic order — cheapest and most likely first

1. **Check the discount/financial-modifier item's actual configured price.** For a "100% off" or "free item" discount, query its real price, not its name (an item named "...100%" was priced `99.99`; Query 2). **A price just under 100 may be deliberate: do not "fix" it to `100.00`.** PM rule (2026-08-03): every tenant needs a minimum billable amount for electronic invoicing, platform-wide (Argentina $0.01, Peru $0.30, Uruguay $0.01), so a sale cannot settle at exactly $0.00. Setting the discount to 100.00 would produce the $0.00 sale the rule forbids. The real defect is that a flat percentage leaves `subtotal × 0.0001`, which drifts from the intended fixed floor as the bill grows ($0.14 and $0.17 against a $0.01 floor). The fix (a fixed-amount-off-remainder mechanism, or a validator/POS tolerance for a residual at the country floor) is a business/dev decision; do not propose the price change as the fix.
2. **Replay the real formula with real numbers in a small mockup before calling anything confirmed.** With clean whole-dollar prices the shortfall of a percentage discount is exactly `subtotal × (100 - discount%) / 100`, which makes "plausible" exact to the cent. Trust only its residual-amount prediction, not its pass/fail one, until the validation total is modeled ("Two totals").
3. **Test the same combination on a plain-item line vs a combo-structured line** of the same product (e.g. "Franui" vs "Franui c/Combo"). A residual invisible on combo lines but hard-failing on plain lines points at the two-totals divergence; confirming why needs the real payload.
4. **Payment method is very likely a red herring: test it.** `ValidateAmount`'s client-side mismatch handling self-heals only a payment line literally named "Efectivo". In the confirmed case cash failed like a bank transfer: the mismatch was a real amount owed.
5. **If the item is promotion-linked, check the `Promotions` window** (`ValidSinceDate`/`ValidToDate` vs `ActivatedDate`/`DeactivatedDate`) as a parallel check only: an expired-but-activated or valid-but-never-activated promotion is a separate bug class; confirm it is in the failing sale's path first.
6. **The failed sale is not in SQL, and not in Cosmos either.** `Sale`/`SaleDetail` are CosmosDB documents (Sales service) and the validator throws *before* `_saleRepository.AddAsync`, so a rejected sale never persists. See below.

## Getting real per-line data (`UnitPrice`/`Quantity`/`Type`/`TypeDetail`)

**Do not start with logs for the exception content.** `SaleService.CreateAsync`'s catch only does `_log.LogError(null, e, e.Message)`. Sales' `AppServiceAppLogs` in Graylog hold only ASP.NET Core pipeline tracing (`Start/End processing HTTP request`, `Executed action method`; same as `20260728_logging-verbosity-ef-core-cors`) plus a generic Kestrel `"An unhandled exception was thrown by the application."` line with `message: null` in its `properties`: neither the exception text nor the payload is captured. Confirm this once per app/tenant and stop iterating search phrases.

**Two traps when reading these logs:**
- `"...Validation state: Valid"` refers to ASP.NET Core model-binding/DataAnnotations of the DTO shape, not to `SaleCreateCmdValidator`; it does **not** mean `payments >= total` passed.
- A generic "unhandled exception" near your test time may not be yours: check the surrounding lines of the same request. In WEISS two such entries were an unrelated transient `AddFriendlyCode` failure (the preceding `"Generando FriendlyCode para Sale..."` line only runs *after* FluentValidation passed).

**The better path: query CosmosDB for a *passing* sale.** A successful sale persists, so pull the real `Details[]` of an accepted test sale on the same POS terminal (e.g. the combo+discount case that passed): ground-truth `Type`/`TypeDetail`/`UnitPrice`/`Quantity`/`Total` with no logging gaps.

**Tenant attribution caveat (cost significant time in WEISS):** `SMARTFRAN-CLOUD-SALES-PRO` is one shared App Service for every tenant and its Graylog logs carry **no tenant-identifying field** (`resourceId`/`name`/`category` are always the shared-app values). A `FranchiseeCode`/`FranchiseCode`/`PosCode` triplet read from a log line, even from a confirmed-200 `Sale/Create`, **cannot be assumed to belong to the tenant under investigation**: in WEISS two Cosmos queries on genuinely persisted sales returned zero rows because the codes almost certainly belonged to another tenant's interleaved traffic.

Ways to get the right triplet, in order of preference:
1. Ask whoever runs the POS test for the exact `FranchiseeCode`/`FranchiseCode`/`PosCode` from the browser network tab (the `SaleCreateCmd` request body) during a live test: unambiguous, and in practice often the only reliable option.
2. **Tried and ruled out in WEISS; do not repeat without new information:** `Business_<TENANT>` has `Franchise`, `Franchisee`, `SalePoint` (all **singular** table names) and `RelatedCode` (opaque codes keyed by `FranchiseId`/`FranchiseeId`/`SalePointId`, joined to `RelationCode`), but every WEISS `RelatedCode` row was Uruguay fiscal/e-invoicing data (`FE-PVTA`, `FEUY-IDCOMERCIO`, `FEUY-APIKEY`, ...) and the `SalePoint`-level lookup returned zero rows: the Sales/Cosmos routing codes are not reachable in this SQL schema. They may live in the `Person`/Company cloud identity system (`Franchise.PersonCloudId`/`CompanyCloudId`), unexplored.
3. Last resort, the Graylog technique below, verified independently (does the resulting Cosmos query return *any* row?) before trusting the values.

When searching Franchise names in `Business_<TENANT>` expect several candidates (WEISS returned test franchises and another location, `Weiss Ensenada`, next to the target `Weiss Punta del Este`): match the specific location name.

Graylog technique: `SaleService.AddFriendlyCode`'s `_log.LogInformation($"Generando FriendlyCode para Sale. FranchiseeCode: {...}, FranchiseCode: {...}, PosCode: {...}, ShiftNumber: {...}")` fires on every attempt, success or failure; treat its values as unverified until a Cosmos query confirms them.
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
Field names: the event's real timestamp is **`time`** (ISO 8601); `timestamp` is Graylog's receive time in a custom format that throws `parse_exception` on a hand-typed literal (use relative expressions like `now-2d`). Pull one sample document first (`{"query":{"match_all":{}}, "sort":[{"time":{"order":"desc"}}], "size":1}`).

Before suspecting the partition-key values when a query comes back empty, rule out the cheaper causes: confirm the container name from source (`grep -rn "GetContainer(" .../CosmosSDK/*Repository.cs`; for Sales `cosmosConnection.GetContainer("Sales")`), and that the entity has no `[JsonProperty]` override and the Cosmos client options set no camelCase serialization policy (both true for `Sale.cs`/`SaleContext.cs` at the time). Only then is tenant misattribution the leading explanation of a confirmed-200-but-zero-rows result.

**Confirm the Cosmos account, database and container** for the tenant (account name first: `cloud/docs/infrastructure.md` → "Core Shared Services"; it does not follow the plain `smartfran-cloud-cosmos-<tenantId>` pattern, do not guess it):
```bash
az cosmosdb sql database list --account-name <account> --resource-group SmartFran.Cloud.PRO.<TENANT> -o table
az cosmosdb sql container show --account-name <account> --database-name Sales-<TENANT> \
  --name Sales --resource-group SmartFran.Cloud.PRO.<TENANT> --query "resource.partitionKey"
```
**Query the `Sales` container in the Portal's Data Explorer** (no `az` command queries SQL API documents; `az cosmosdb sql *` manages metadata only):
```sql
SELECT c.id, c._ts, c.ShiftNumber, c.FriendlyCode, c.Details, c.MoneyMovementData
FROM c
WHERE c.FranchiseeCode = "<value>" AND c.FranchiseCode = "<value>" AND c.PosCode = "<value>"
ORDER BY c._ts DESC
```

## Databases and confirmed table/column names

Two SQL databases per tenant (elastic pool `t102-smartfran-cloud-<tenant>`): using the wrong one is the most common mistake. They are **Azure SQL Database**: `USE [OtherDatabase];` is a silent no-op, so "Invalid object name" for a table you know exists means the tool is connected to the other database (`SELECT DB_NAME();`); switch it in the tool's connection picker, not with SQL.

**`SmartFran.Cloud.Business_<TENANT>`** — promotion/combo definitions:

| Table | Key columns |
|---|---|
| `Promotions` | `Name`, `PromotionType` (0=Promotion, 1=Combo), `ValidSinceDate`/`ValidToDate`, `ValidSinceMinute`/`ValidToMinute`, day-of-week bools, `MandatoryForAll`, `ActivatedDate`/`DeactivatedDate` |
| `PromotionGroups` | `PromotionId` (FK), `Amount`, `Type` (string: FixedPrice/FixedDiscount/PercentDiscount/PriceList), `HasAdditionals`, `MultipleSelection` |
| `PromotionDetails` | `PromotionGroupId` (FK), `ArticleId`: eligible articles per group |
| `PromotionApplies` | `Include`, `FranchiseId`, `FranchiseeId`, `PriceListId`, etc. |
| `PromotionGroupsApplies` | note the name ("GroupsApplies"). **`PromotionValue`** is a *third*, independent pricing source (below) |
| `Oversales` / `OversaleComboPromotionItems` | Upsell-prompt config; usually a distractor (governs cashier-facing upsell offers, not manually added combo contents) |

**`SmartFran.Cloud.Catalog_<TENANT>`** — item/price/financial-modifier definitions:

| Table | Key columns |
|---|---|
| `Items` | `Name`, `GroupId`, `ForSale` |
| `Groups` | `FinancialModify` (0=NoApply, 1=Discount, 2=Surcharge, 3=Delivery): how a "discount item" is told from a product |
| `PriceLists` / `PriceDetails` | `PriceDetails.ItemId`/`PriceListId`, **`PublishedPrice`/`NewPrice`** (there is no `Price` column), `Enabled` |

**An article can have up to three different prices:**
1. `PriceDetails.PublishedPrice`/`NewPrice` (Catalog): the standalone retail price.
2. `PromotionGroupsApplies.PromotionValue` (Business), where `PromotionGroups.Type = 'FixedPrice'`: the price of the same article as a component of a specific combo/promotion, scoped to a `PriceListId`. In WEISS one article was `9900` standalone and `250` as a combo add-on in the same price list (~40×).
3. An article's eligibility (`PromotionDetails.ArticleId`) can appear across dozens of `Promotions` rows (one article can be an add-on of many unrelated combos): search `PromotionDetails.ArticleId` with the article's `Id` rather than a name search on `Promotions`.

```sql
-- Query 1: locate a promotion/combo by name, and check its activation vs date-window state
SELECT Id, Name, Description, PromotionType, MandatoryForAll,
       ValidSinceDate, ValidToDate, ActivatedDate, DeactivatedDate
FROM Promotions
WHERE Name LIKE '%<term>%' OR Description LIKE '%<term>%';

-- Query 2: locate a discount/financial-modifier item and its REAL standalone price
-- (run against Catalog_<TENANT>; confirm with `SELECT DB_NAME()` first if unsure)
SELECT i.Id, i.Name, i.GroupId, g.FinancialModify, g.Name AS GroupName
FROM Items i
LEFT JOIN Groups g ON g.Id = i.GroupId
WHERE i.Name LIKE '%<term>%';

SELECT pd.ItemId, pd.PriceListId, pd.PublishedPrice, pd.NewPrice, pd.Enabled
FROM PriceDetails pd
JOIN Items i ON i.Id = pd.ItemId
WHERE i.Name LIKE '%<term>%';

-- Query 3 (run against Business_<TENANT>): every combo/promo that prices this article as a
-- component, and at what value per price list — use the article's Id from Query 2
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

**CosmosDB (Sales service, SQL API)** holds the actual sales: database `Sales-<TENANT>`, container `Sales`, partition key `[FranchiseeCode, FranchiseCode, PosCode]`. Only **passing** sales appear there. The full container list is in `cloud/docs/infrastructure.md` (authoritative).

## Marketplace catalogs are a different system

If the bug comes from a **marketplace/delivery-platform app screen** (PedidosYa, Rappi, UberEats, ...; mobile-app-style UI, not the internal Blazor POS admin), do not assume it derives from `Business_<TENANT>`/`Catalog_<TENANT>`. Confirmed in WEISS `20260816_clasica-dijon-invalid-sale-weiss` (a combo showing 3 burger options instead of 2 on PedidosYa's app):

- `repo/SmartFran.Cloud` has **no menu/catalog export to any marketplace**: `grep -rli "pedidosya"` finds only order-*receiving* logging enums (`PedidosYaOrderPhases`/`PedidosYaOrderSteps`/`PedidosYaOrder`), nothing pushing `PromotionGroups`/`PromotionDetails` outward.
- `smartpedidos/` (repos `dev-scr-smartPedidos-platformsService`, `dev-src-smartPedidos-concentradorService`) has a real PedidosYa integration, but order-lifecycle only (`receiveOrder`, `confirmOrder`, `branchRejectOrder`, `dispatchOrder`, parsing an *incoming* order's `selectedToppings`); `grep -rli "menu"` across both returns **zero results**.
- So a marketplace app's menu is configured on the platform's merchant portal, independent of `Business`/`Catalog` and of SmartPedidos. Verify the source screen first (ask for a screenshot) before ruling Business/Catalog out, since internal and marketplace screens can show similar symptoms.
- Before concluding "marketplace-side", confirm the *internal* side is clean: a Business/Catalog DB query and a trace of `DialogBuildCombo.razor.cs`'s `GetItems(groupId)`/`GetPriceDetails(groupId)`, which build each group's options strictly from that group's own `PromotionDetails.ArticleId` rows.

## Source branch

`repo/SmartFran.Cloud` tracks `origin/dev`, which lags and diverges from `origin/main` (the PRO release branch). Before treating a source trace as applicable to a production bug, fetch and diff the files you read against `origin/main` (`git diff origin/main origin/dev --stat -- "<file>"`) and read the `origin/main` version if they differ. Detail and evidence: `cloud/CLAUDE.md`.

## Worked examples

- `cloud/events/20260802_promocion-invalida-weiss-franui/` (`_investigation.md` narrative, `_scripts.py` runnable mockup): a genuine `SaleIsInvalid` in WEISS (Uruguay). The item "Descuento Influencers 100%" (Catalog `Items.Id=430`, `Groups.FinancialModify=1`) was priced `99.99` in all 8 `PriceDetails` rows, leaving `subtotal × 0.0001` (reproduced $0.14 / $0.17 to the cent). Plain "Franui" (`Id=386`) failed, "Franui c/Combo" (`Id=423`) passed: combo body lines are re-rounded (`UnitPrice = Round(Total/Quantity, 2)`), which swallows the residual silently (so every combo sale with that discount under-collects 0.01 % unnoticed). The 99.99 turned out to be deliberate (minimum-invoicing rule above), so the originally proposed fix (set 100.00) is wrong and was withdrawn. **Not fully confirmed:** the exact plain-vs-combo mechanism (validation total vs display total is well supported by source, not closed with a real payload). Secondary, unrelated: promotions 183/186 ("Ruleta Articulo Bonificado") activation gap.
- `cloud/events/20260816_clasica-dijon-invalid-sale-weiss/`: not a rejection but a combo *display* defect reported from PedidosYa. `Business_WEISS`/`Catalog_WEISS` clean, internal POS combo builder correct (screenshot + source trace), cause isolated to PedidosYa's own menu configuration (external). Established the "Marketplace catalogs" section.

## Constraints

- Read-only by default. Any `UPDATE`/`DELETE` on Catalog/Business data is a production write: confirm with the owner of that tenant's pricing/promotions before proposing it as final, and show a `SELECT` preview of the affected rows first.
- Output SQL as copy-paste blocks; the user runs it and pastes results back. Never execute directly.
- Content written to `cloud/events/` follows that project's Spanish/English split (`cloud/CLAUDE.md`).
