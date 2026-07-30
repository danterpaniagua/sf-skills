# Skill: Fraud Detection — Member Dispute

## Scope

A member disputes specific redemptions/deductions on their **own account** and asks "what happened to my points" — this is a single-`CustomerId` balance reconciliation, **not** a network investigation. No hub/relay/exploit search, no memory update to `known_hubs.md`/`known_relays.md` unless the trace itself surfaces a fraud pattern.

> Network-wide investigations (transfer fraud, accumulation anomaly, manual assignment exploit) are handled by the **`fraud-points`** skill. If tracing this dispute surfaces cross-branch POS activity or a franchise-network pattern, pivot to **`fraud-pos`**.

This skill operates on the **production database** (`SmartFran.Solution.SmartLoyalty`). Access is **implicit within this skill's scope** — no explicit user request required.

Shared reference material (not duplicated here — read from `fraud-points` when needed):
- Core identity tables (`sml.Person`, `sml.CustomerPointsLog`, `smlst.CustomerPointsLog`) — see `loyalty-fraud-points.md` Core Tables.
- Points accounting format (Axis 1 origin + Axis 2 four-state) — see `loyalty-fraud-points.md` "Points accounting — required in every analysis output".
- Full `EventTypeCode` reference — `loyalty/docs/eventtypecode_reference.md`.

---

## Core Tables (dispute-specific)

### `Sml.Sale`
`Id, SaleDate, Delivery, BranchOfficeId, CustomerId, CustomerCardId, PointsLogId, InvalidatedPointsLogId, PaymentTypeCode, Distribution, PlatformCode`

One row per POS/platform transaction. Confirmed 2026-07-11 while tracing a member dispute.

| Column | Fraud/dispute relevance |
|---|---|
| `BranchOfficeId` | FK → `sml.BranchOffice.Id` — join with `bo.Code`/`bo.Name` to compare against the customer's known/home branch. A disputed sale at a foreign branch is a stronger signal than one at the customer's regular branch. |
| `CustomerCardId` | Physical/digital card identifier. **A different `CustomerCardId` across disputed vs. undisputed sales is the clearest card-cloning signal.** Identical card ID across all sales rules out cloning/substitution. |
| `PaymentTypeCode` | e.g. `EF` (efectivo/cash), `TD` (tarjeta de débito), `MU` (multi/plataforma). Two different payment types on the same card within minutes at the same branch can indicate a second, unrelated transaction was rung up in the same visit. |
| `PlatformCode` | e.g. `POS` (physical terminal), `PG` (payment gateway/app-integrated). Consistent platform across disputed and undisputed sales = normal usage pattern, not remote takeover. |
| `PointsLogId` | FK → `sml.CustomerPointsLog.Id` — direct link from the sale to its resulting point-log row(s). |
| `InvalidatedPointsLogId` | Populated when the sale was later invalidated/reversed — check this before treating a sale as still-active. |

### `Sml.SaleDetail`
`Id, Amount, SaleId, SalePromotionId, ArticleId, PosArticleRef`

Article-level line items for a sale. One `SaleId` can have multiple `SaleDetail` rows (e.g. a combo redemption bundling two articles under one `SalePromotionId`). Join `SaleId` → `Sml.Sale.Id`.

### `Sml.SalePromotion`
`Id (GUID), PromotionId, SaleId`

Links a sale to the specific promotion/catalog item redeemed. Join `PromotionId` → `Sml.Promotion.Id` to decode what was actually purchased/redeemed.

### `Sml.Promotion`
`Id, Name, Description, Points, Quantity, ValidSinceDate, ValidToDate, Type, ...` (132 columns confirmed via `INFORMATION_SCHEMA`, most are scheduling/eligibility flags)

Master catalog of redeemable promotions. **`Points` is negative for redemptions** (e.g. `-5000`) and this is the field that decodes a `DiscountPointsByExchange` event into an actual product name.

```sql
-- Decode what a redemption (PromotionId) actually was
SELECT Id AS PromotionId, Name, Description, Points, Type, Quantity
FROM [SmartFran.Solution.SmartLoyalty].Sml.Promotion
WHERE Id IN (<promotion_id_1>, <promotion_id_2>, ...);
```

Useful in disputes: if two `DiscountPointsByExchange` rows share the same `SaleId` and `PromotionId`, they are the same redemption split into catalog-quantity lines (e.g. 2× a 5,000-pt item = one 10,000-pt redemption). If a disputed deduction has its **own distinct `SaleId`/`PromotionId`** resolving to a **different named product**, it is a genuinely separate transaction — not a system duplicate/glitch.

---

## Investigation Workflow

Confirmed workflow (2026-07-11):

1. **Identity resolution (Q1)**: resolve CustomerId from DNI, confirm account origin (channel/date/branch), email validation, deactivation status — same Axis 1 fields as any actor (see `fraud-points`).
2. **Current balance (Q2)**: `smlst.CustomerPointsLog.Points`.
3. **Full event history (Q3)**: pull `sml.CustomerPointsLog` for a window comfortably wrapping every disputed date — do not query per-event ±2h windows separately, one continuous range is easier to reconcile.
4. **Point-in-time balance — verify, don't infer.** Two independent checks, both required before telling the member anything is confirmed:
   - **Lifetime sum check**: `SELECT SUM(Points) FROM sml.CustomerPointsLog WHERE CustomerId = ...` with no date filter. Must equal the current `smlst` balance exactly. If it doesn't, something adjusts the balance outside the log and the rest of this method is unsafe to use.
   - **Direct historical balance**: `SELECT SUM(Points) FROM sml.CustomerPointsLog WHERE CustomerId = ... AND LogDate < '<cutoff_GMT>'` gives the exact balance as of any past date. **Never derive a historical balance by subtracting known events from the current balance without running this direct sum as a cross-check** — an inferred number is only as good as the assumption that no event was missed, and that assumption must be tested, not asserted.
5. **Running balance table**: once both checks pass, walk forward event-by-event from the verified starting balance through every disputed transaction to the current balance. If the walk lands exactly on the confirmed current balance, there is no undiscovered fourth deduction — the member's tally of "more was taken" is fully explained by the transactions already found (or isn't, and the arithmetic will show a gap that needs another query pass).
6. **Product-level decode (if redemptions are disputed)**: join `SaleId`/`PromotionId` through `Sml.SalePromotion` → `Sml.Promotion` to get the actual product name. Two `DiscountPointsByExchange` rows under the **same** `SaleId`/`PromotionId` are one redemption split into catalog lines (normal). A disputed deduction with its **own** `SaleId`/`PromotionId` resolving to a **different named product** is a real, distinct transaction — not a duplicate/glitch — which shifts the likely explanation toward card-sharing or POS/counter error rather than a backend bug.
7. **Card/branch trace (`Sml.Sale`)**: pull `CustomerCardId`, `BranchOfficeId`, `PaymentTypeCode`, `PlatformCode` for every sale in the window. Identical `CustomerCardId` across disputed and undisputed sales rules out card cloning. Branches/platforms matching the member's own undisputed history rule out remote takeover — points toward shared card use or a counter-side error instead.

This path produces no hub/relay memory update — the member is the disputant, not a suspect, unless the trace itself surfaces a fraud pattern from `fraud-points`.

---

## Evidence Package — Closure (`_ops.md`)

Single-account scope — the three network-wide metrics (Clientes únicos, etc.) collapse to 1 and can be noted as such rather than omitted. Required sections:

1. **Metrics table** — same format as accumulation anomaly (`fraud-points`), scoped to one customer.
2. **EventTypeCode breakdown** — Q1-style, scoped to the account.
3. **Reconstrucción por transacción disputada** — one table per disputed date: hora UTC-3, SaleId, PromotionId, producto decodificado (`Sml.Promotion.Name`), puntos, sucursal, tarjeta, PaymentType. State explicitly whether each disputed line matches (same `SaleId`) or is separate from (different `SaleId`/product) any transaction the member acknowledges.
4. **Balance reconciliation table** — verified starting balance (direct `SUM`, not inferred) → running balance through every event → confirmed current balance. Must close exactly; note explicitly if it doesn't.
5. **Points accounting** — Axis 1 origin + Axis 2 four-state, same format as `fraud-points`. For a dispute, the closure table (`Actor | DNI | Rol | Pts Totales | Recuperable | No Recuperable`) rol is "Socia — denunciante", not hub/relay/feeder.
6. **Acciones propuestas** — split into: verification steps with the branch/customer (not reversals yet) / conditional remediation (credit back if verification fails to find a mundane explanation) / explicitly note "Sin acciones técnicas" if no platform gap was found (distinguishes this from the exploit-closure format, where technical remediation is usually required).

Place under `events/YYYYMMDD_description/`.
