# Skill: Fraud Detection — Points

## Scope

Detect fraudulent activity in point movements within `SmartFran.Solution.SmartLoyalty`: transfers, accumulations, manual assignments, and redemptions.

> POS terminal manipulation via franchise networks is handled by the **`fraud-pos`** skill. When an account shows high-volume cross-branch POS activity or geographic impossibility, note the signal and invoke `fraud-pos` for the franchise investigation path.

This skill operates on the **production database** (`SmartFran.Solution.SmartLoyalty`). Access is **implicit within this skill's scope** — no explicit user request required.

---

## Core Tables

### `sml.CustomerPointsLog`
`Id, CustomerId, LogDate, Note, Points, EventTypeCode, SaleId, PromotionId, ArticleId, ManualAssignPointsId`

Primary log of all point events. Known `EventTypeCode` values:

| EventTypeCode | Meaning |
|---|---|
| `PointsByTransferSent` | Points leaving the account — sender side |
| `PointsByTransferReceived` | Points arriving at the account — receiver side |
| `EarnPointsByBuying` | Points earned from a POS purchase (`SaleId` + `ArticleId` populated) |
| `EarnPointsByPromotion` | Promotional bonus tied to a purchase (`PromotionId` + `SaleId` populated; can be 0 pts) |
| `Article99999WithoutPoints` | Purchase of a zero-point article (`SaleId` populated, always 0 pts) |
| `CompensationalPoints` | Manual administrative compensation (`ManualAssignPointsId` populated, no `SaleId`) |
| `DiscountPointsByExchange` | Points deducted in a redemption/exchange event (negative value) |
| `NewCustomer` | Welcome grant on account creation (typically 0 pts) |
| `RemovePointsBySaleInvalidation` | Points reversed due to cancelled sale (negative value) |

A transfer is a **paired operation** written atomically:

| Column | Sender row | Receiver row |
|---|---|---|
| `EventTypeCode` | `PointsByTransferSent` | `PointsByTransferReceived` |
| `Points` | negative (e.g. `-50`) | positive (e.g. `+50`) |
| `LogDate` | identical to the microsecond | identical to the microsecond |
| `Id` | N | N + 1 (always sequential) |
| `SaleId / PromotionId / ArticleId / ManualAssignPointsId` | all NULL | all NULL |

Pairing key: **`sml.PointsTransference`** — join on `IdCustomerPointsLogSender` / `IdCustomerPointsLogReceiver`. `LogDate` match and sequential `Id` (N / N+1) are secondary confirmation. `ABS(sender.Points) = receiver.Points`.

### `sml.ManualAssignPoints`
`Id, RegisterByUser, AssignmentConcept, Points, Status, AssignDate, ErrorLog, Catalog_Id`

Administrative table — one row per manual point grant. FK from `sml.CustomerPointsLog.ManualAssignPointsId`.

| Column | Fraud relevance |
|---|---|
| `RegisterByUser` | Operator who created the grant — no ceiling or approval gate exists |
| `AssignmentConcept` | Grant type — see channel classification below |
| `Points` | No technical maximum — any value is accepted and auto-approved |
| `Status` | Always `Approved` — no second-level review implemented |
| `Catalog_Id` | Reason code reference — `NULL` in all known exploit cases; absence means no justification required |
| `ErrorLog` | `NULL` on all successful grants — useful as negative confirmation only |

**AssignmentConcept channel classification:**

| Concept | Channel | Recipients | Investigation status |
|---|---|---|---|
| `CompensationalPoints` | Customer service compensation / HR employee grants | Club Grido end customers; also Grido employees via `GSFERNANDEZ` | **Exploit confirmed for unknown operators** — no ceiling, no Catalog_Id. `GSFERNANDEZ` grants are legitimate HR employee compensation — exclude from exploit totals. |
| `HumanResourcesPoints` | Grido HR | **Grido employees only** — flag any non-employee | Legitimate — exclude from exploit totals |
| `InstitutionalPoints` | Institutional / internal | **Grido employees** (from / to) | Legitimate — exclude from exploit totals |
| `PrizePoints` | Prize / contest | End customers | Active continuously since 2023 via `GS31087232` — not discontinued |

**Known operators:**

| Operator | Channels | Role | Exclude from exploit? |
|---|---|---|---|
| `GSFERNANDEZ` | `CompensationalPoints` | HR crew — assigns compensation points to Grido employees | Yes — grants are legitimate employee compensation |
| `GS31087232` | `PrizePoints`, `InstitutionalPoints`, `HumanResourcesPoints` | Long-tenure operator — prize batches, institutional grants, HR payroll | Partially — HR/Institutional legitimate; PrizePoints batches require per-event review (high-value uniform batches flagged Jun 2025, Jun 2026) |
| `GSLCEBALLOS` | `PrizePoints`, `HumanResourcesPoints`, `CompensationalPoints`, `InstitutionalPoints` | HR crew — all four channels; also performs negative grants (reversals) to claw back fraudulent balances | Yes — all channels legitimate; negative grants are protective reversals, not exploits. Confirmed reversal: Eustacio Castro −62,475 pts (2024-12-20) |

**Operator username convention:** Usernames are uppercase (e.g. `GSFERNANDEZ`, `GSLCEBALLOS`, `GS31087232`). Use case-insensitive comparison if querying `RegisterByUser`.

**Exploit vector (confirmed 2026-06-02):** `sml.ManualAssignPoints` has no row-level ceiling, no mandatory `Catalog_Id`, and no approval workflow. A single operator can inject unlimited points to any customer with no technical control. Use Q_E1–Q_E5 to quantify historical exposure when this pattern is detected.

Auxiliary tables in the same schema (confirmed via `INFORMATION_SCHEMA`):
- `Sml.AssignmentPointsLog`
- `Sml.AssignmentPointsErrorLog`
- `Sml.ManualExchangeEnabled`
- `Sml.PreAssignedGiftRef`

---

### `sml.Customer`
`Id, FormDate, CreatedDate, UserId, EnrolledId, RegisterById, Country_Id, RegistrationChannel`

> `DeactivatedDate` does NOT exist on this table — confirmed SQL error 207 on two separate investigations (2026-06-03, 2026-06-12). Do not reference it in queries.

**Deactivated account detection:** Upon deactivation, the platform obfuscates all PII in `sml.Person` — `FirstName`, `LastName`, `Email`, `UidSerie`, and other identity columns are replaced with the account's CustomerId GUID. Detect deactivation exclusively from `sml.Person` using:

```sql
-- Deactivated: UidSerie is a GUID, fails TRY_CAST to BIGINT
CASE WHEN TRY_CAST(p.UidSerie AS BIGINT) IS NULL
     THEN 'Desactivada' ELSE 'Activa' END AS EstadoCuenta
```

A `Person` row where `UidSerie` (or `FirstName` / `LastName`) contains a GUID string (e.g. `79C680B6-5EAC-C58D-D81E-08DD68DA7039`) is a deactivated and anonymized account — not a ghost registration. No join to `sml.Customer` is needed for deactivation checks.

Points transferred to a deactivated account may be frozen depending on platform behavior.

Relationship: `sml.CustomerPointsLog.CustomerId → sml.Customer.Id` (many-to-one).

| Column | Fraud relevance |
|---|---|
| `CreatedDate` | Cluster of accounts created in same window = coordinated bulk registration |
| `RegistrationChannel` | `PUNTO DE VENTA` = registered at a POS terminal; `APP` / `WEB` = self-registered. Same channel across many mule accounts = automated registration. |
| `RegisterById` | **FK → `sml.BranchOffice.Id`** (confirmed 2026-06-12). Identifies the branch where the account was created. `EnrolledId` is NULL when no enrolment branch was captured (older POS registrations). Same `RegisterById` across multiple accounts = same terminal = insider risk. |
| `EnrolledId` | FK relationship unconfirmed. May be NULL for older registrations. |

### `smlst.CustomerPointsLog`
`CustomerId, Points, LastLogDate`

Summary table — **current point balance per customer**. One row per customer, updated on every point event.

| Column | Fraud relevance |
|---|---|
| `Points` | Current balance. Receiver balance >> transfer received = pre-existing stock (consolidation). Sender balance = 0 = full account drain. |
| `LastLogDate` | Should match the last transfer `LogDate`. Mismatch signals additional activity not yet investigated. |

Use this table for fast balance checks on hub/receptor accounts without scanning the full log. Join on `CustomerId = sml.Person.Id`.

### `Sml.LocationAttributeValue`
`Id, LocationId, AttributeCode, Value`

Application configuration table. Stores the **active business limits** used by the platform. Always query this table to confirm current thresholds before flagging a limit breach — values may change over time.

```sql
SELECT AttributeCode, Value
FROM [SmartFran.Solution.SmartLoyalty].Sml.LocationAttributeValue
WHERE LocationId = 1
ORDER BY AttributeCode;
```

**Current configured limits (LocationId = 1 — global config):**

| AttributeCode | Value | Meaning |
|---|---|---|
| `PointsTransferActive` | 1 | Transfers enabled |
| `CustomerPointsMinLimit` | 3,000 | Customer — daily accumulation cap |
| `CustomerPointsMidLimit` | 5,000 | Customer — weekly accumulation cap |
| `CustomerPointsMaxLimit` | 10,000 | Customer — monthly accumulation cap |
| `CustomerPointsMinLimitTransfer` | 8,000 | Customer — daily transfer cap |
| `CustomerPointsMidLimitTransfer` | 10,000 | Customer — weekly transfer cap |
| `CustomerPointsMaxLimitTransfer` | 13,000 | Customer — monthly transfer cap |
| `ColaboratorPointsMinLimitTransfer` | 30,000 | Collaborator (employee) — daily transfer cap |
| `ColaboratorPointsMidLimitTransfer` | 40,000 | Collaborator (employee) — weekly transfer cap |
| `ColaboratorPointsMaxLimitTransfer` | 60,000 | Collaborator (employee) — monthly transfer cap |

> **Naming convention:** `Min` = daily, `Mid` = weekly, `Max` = monthly.  
> **LocationId 9** has a secondary accumulation config (same 3k/5k/10k values, no transfer limits defined).

**Collaborator vs Customer distinction — critical for limit validation:**

| Profile | Identified by | Daily transfer cap | Weekly | Monthly |
|---|---|---|---|---|
| **Customer** | No `HumanResourcesPoints` in history; regular POS activity | 8,000 | 10,000 | 13,000 |
| **Collaborator (employee)** | Regular `HumanResourcesPoints` grants from "Colaboradores" lists | **30,000** | **40,000** | **60,000** |

A transfer flagged as a limit breach must be re-evaluated against the sender's actual profile. A Collaborator sending 30,000 pts in one day is at the daily cap, not over it. A Customer sending the same amount is 3.75× over their daily cap.

### `sml.PointsTransference`
`Id, IdCustomerPointsLogSender, IdCustomerPointsLogReceiver, Date, SourceChannel`

One row per transfer. Links the sender and receiver `CustomerPointsLog` rows directly — use this as the primary pairing key instead of the `LogDate` match approach.

| Column | Description | Fraud relevance |
|---|---|---|
| `IdCustomerPointsLogSender` | Sender `CustomerPointsLog.Id` | Direct FK to sender row — always `IdCustomerPointsLogReceiver - 1` |
| `IdCustomerPointsLogReceiver` | Receiver `CustomerPointsLog.Id` | Direct FK to receiver row — always `IdCustomerPointsLogSender + 1` |
| `Date` | `DATETIMEOFFSET +00:00` (GMT) | Primary filter for time-window queries |
| `SourceChannel` | `WEB` or `APP` | Uniform channel across many transfers = automation signal |

### `Sml.Sale`
`Id, SaleDate, Delivery, BranchOfficeId, CustomerId, CustomerCardId, PointsLogId, InvalidatedPointsLogId, PaymentTypeCode, Distribution, PlatformCode`

One row per POS/platform transaction. Confirmed 2026-07-11 while tracing a member dispute — not previously documented.

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

### `sml.Person`
`Id, FirstName, LastName, GenderCode, BirthDate, Email, UidSerie, UidCode, AddressId, UpdateDate`

Customer identity. Relationship: `sml.CustomerPointsLog.CustomerId → sml.Person.Id` (many-to-one).

| Column | Meaning | Fraud relevance |
|---|---|---|
| `Id` | GUID — joins to `CustomerPointsLog.CustomerId` | Primary join key |
| `UidCode` | Document type (e.g. `Dni`) | Identity type |
| `UidSerie` | Document number | `UidCode` + `UidSerie` + `Country_Id` is the natural identity key; duplicate = multiple accounts per person |
| `Email` | Contact email | Shared email across accounts = duplicate account signal |
| `UpdateDate` | `DATETIMEOFFSET` | Sudden profile update before a transfer = tampering signal |

**Bulk CustomerId resolution (Q_RESOLVE_CUSTOMERIDS):** When multiple actors have pending CustomerId values, resolve all at once rather than one by one:

```sql
SELECT
    TRY_CAST(p.UidSerie AS BIGINT)   AS DNI,
    p.FirstName + ' ' + p.LastName   AS Name,
    p.Id                              AS CustomerId
FROM [SmartFran.Solution.SmartLoyalty].sml.Person p
WHERE TRY_CAST(p.UidSerie AS BIGINT) IN (
    <dni_1>, <dni_2>, ...
)
ORDER BY TRY_CAST(p.UidSerie AS BIGINT);
```

Deactivated actors will not appear (their `UidSerie` is a GUID, `TRY_CAST` returns NULL). For those, `CustomerId` is already the only identifier available from prior investigation.

---

## Platform scope — Multi-brand

SmartLoyalty hosts **multiple franchise brands** on the same instance. Confirmed brands:

| Brand | Registration identifier | Notes |
|---|---|---|
| Club Grido | `GRIDO*`, `GR*` branch codes | Primary brand — majority of POS network |
| Mundo Helado | `MDOH*` branch codes | Ice cream franchise; separate franchise operator |

Fraud crosses brand boundaries — an actor registered under Mundo Helado (e.g. branch MDOH01) may receive or spend points within the Club Grido network. Do not limit investigation scope to a single brand when following transfer chains.

---

## Business Rules (Club Grido)

Limits are stored in `Sml.LocationAttributeValue` (LocationId = 1) and differ by account type. The platform does **not** enforce any of these in real time — confirmed on 2026-05-14/15 (accumulation) and 2026-06-03 (transfer).

### Accumulation limits (both Customer and Collaborator)

| Period | AttributeCode | Limit |
|---|---|---|
| Daily | `CustomerPointsMinLimit` | **3,000 pts** |
| Weekly | `CustomerPointsMidLimit` | **5,000 pts** |
| Monthly | `CustomerPointsMaxLimit` | **10,000 pts** |

### Transfer limits — Customer (regular end user)

| Period | AttributeCode | Limit |
|---|---|---|
| Daily | `CustomerPointsMinLimitTransfer` | **8,000 pts** |
| Weekly | `CustomerPointsMidLimitTransfer` | **10,000 pts** |
| Monthly | `CustomerPointsMaxLimitTransfer` | **13,000 pts** |

### Transfer limits — Collaborator (Grido employee)

| Period | AttributeCode | Limit |
|---|---|---|
| Daily | `ColaboratorPointsMinLimitTransfer` | **30,000 pts** |
| Weekly | `ColaboratorPointsMidLimitTransfer` | **40,000 pts** |
| Monthly | `ColaboratorPointsMaxLimitTransfer` | **60,000 pts** |

> **Always determine account type before declaring a transfer limit breach.**  
> Collaborator identity is confirmed by recurring `HumanResourcesPoints` grants  
> with notes like "Colaboradores Permanentes" or "Colaboradores Temporales".  
> Source limits: `Sml.LocationAttributeValue` — query before assuming values.

---

## Investigation Workflow

**Step 0a:** Read `loyalty/memory/known_hubs.md` and `loyalty/memory/known_relays.md`. Any CustomerId or DNI from those files appearing in the current investigation must be flagged immediately as a **known actor** before any query is run.

**Step 0b:** Convert user's time window (default UTC-3) to GMT. Query window = event ±2h.

**Step 0c:** As participant identities are resolved (Q4 / identity resolution), apply the **Name & Identity Flags** rules below to every row. Flag suspicious names inline in the analysis — do not wait until the end.

### Transfer fraud

Filter on `sml.PointsTransference.Date`.

1. **Volume check**: count and sum via `sml.PointsTransference`, grouped by calendar day (UTC-3). Join `sml.CustomerPointsLog` on `IdCustomerPointsLogSender` / `IdCustomerPointsLogReceiver`.
2. **Fan-in detection**: `CustomerId` values receiving from multiple senders in the window.
3. **Identity resolution**: join `sml.Person` → name, DNI, email, role (Hub / Receptor / Emisor). Apply Name & Identity Flags to every participant.
4. **Limit validation**: flag senders with `ABS(Points)` > 8,000 pts in a single day.
5. **Hub activity**: balance via `smlst.CustomerPointsLog`. Sender = 0 = full drain. Check post-event activity for canjes or redistribution.
6. **Email verification**: join `sml.Person` → check verified status per the Email Verification schema below. Unverified email on a high-volume sender or receiver = elevated risk signal.
7. **Registration pattern**: group by `RegistrationChannel`, `RegisterById`, `CreatedDate`.

### Manual assignment exploit

When `ManualAssignPointsId IS NOT NULL` is the source of anomalous points — pivot to `sml.ManualAssignPoints` directly. This path bypasses POS and promotion logic entirely.

1. **Operator identification**: `GROUP BY RegisterByUser` on `sml.ManualAssignPoints`. A single operator with high volume is the primary signal.
2. **Breach count (Q_E2)**: count grants > 3,000 pts (daily cap) and > 10,000 pts (monthly cap) in a single assignment. Also compute total excess above daily cap.
3. **Top beneficiaries (Q_E3)**: `GROUP BY CustomerId` — join `CustomerPointsLog` on `ManualAssignPointsId`, then `Person` for identity. Sort by `Puntos_Total DESC`.
4. **Monthly escalation (Q_E4)**: `GROUP BY FORMAT(AssignDate, 'yyyy-MM')` — volume, breach counts, max grant per month. A rising trend with no control event = systemic exploit.
5. **Per-customer monthly breach (Q_E5)**: CTE aggregating per-customer per-month totals; filter `> 10,000`. Reveals compound monthly-cap violations across multiple smaller grants.

---

### Accumulation anomaly

When the spike is **not a transfer** — filter `sml.CustomerPointsLog.LogDate` directly. Exclude `PointsByTransferSent` / `PointsByTransferReceived`.

1. **EventType breakdown (Q1)**: `GROUP BY EventTypeCode` — count and sum. Identifies what type of insertion is driving the spike.
2. **Baseline (Q2)**: same hour on prior 7 days. Confirms whether volume is anomalous vs. historical norm.
3. **Cadence (Q3)**: group by minute. Irregular spread = normal; burst in 1–2 minutes = automation signal.
4. **Top recipients (Q4)**: count and sum per `CustomerId` × `EventTypeCode`. Fan-in signal.
5. **Source analysis (Q5)**: group by `PromotionId` / `ArticleId` / `SaleId` / `ManualAssignPointsId`. Identifies exploited promotion or article.
6. **Manual assignments (Q6)**: filter `ManualAssignPointsId IS NOT NULL`. Any hit = insider risk.
7. **Daily limit check (Q7)**: flag customers with `SUM(Points) > 3,000` for the full UTC-3 day.
8. **Top earner history (Q8)**: weekly/monthly totals for high-volume recipients. Compare against limits and prior session baseline.

---

### Member dispute (single-account balance reconciliation)

When a member disputes specific redemptions/deductions and asks "what happened to my points" — this is not a network investigation (no hub/relay/exploit search). Scope is one `CustomerId`. Confirmed workflow (2026-07-11):

1. **Identity resolution (Q1)**: resolve CustomerId from DNI, confirm account origin (channel/date/branch), email validation, deactivation status — same Axis 1 fields as any actor.
2. **Current balance (Q2)**: `smlst.CustomerPointsLog.Points`.
3. **Full event history (Q3)**: pull `sml.CustomerPointsLog` for a window comfortably wrapping every disputed date — do not query per-event ±2h windows separately, one continuous range is easier to reconcile.
4. **Point-in-time balance — verify, don't infer.** Two independent checks, both required before telling the member anything is confirmed:
   - **Lifetime sum check**: `SELECT SUM(Points) FROM sml.CustomerPointsLog WHERE CustomerId = ...` with no date filter. Must equal the current `smlst` balance exactly. If it doesn't, something adjusts the balance outside the log and the rest of this method is unsafe to use.
   - **Direct historical balance**: `SELECT SUM(Points) FROM sml.CustomerPointsLog WHERE CustomerId = ... AND LogDate < '<cutoff_GMT>'` gives the exact balance as of any past date. **Never derive a historical balance by subtracting known events from the current balance without running this direct sum as a cross-check** — an inferred number is only as good as the assumption that no event was missed, and that assumption must be tested, not asserted.
5. **Running balance table**: once both checks pass, walk forward event-by-event from the verified starting balance through every disputed transaction to the current balance. If the walk lands exactly on the confirmed current balance, there is no undiscovered fourth deduction — the member's tally of "more was taken" is fully explained by the transactions already found (or isn't, and the arithmetic will show a gap that needs another query pass).
6. **Product-level decode (if redemptions are disputed)**: join `SaleId`/`PromotionId` through `Sml.SalePromotion` → `Sml.Promotion` to get the actual product name. Two `DiscountPointsByExchange` rows under the **same** `SaleId`/`PromotionId` are one redemption split into catalog lines (normal). A disputed deduction with its **own** `SaleId`/`PromotionId` resolving to a **different named product** is a real, distinct transaction — not a duplicate/glitch — which shifts the likely explanation toward card-sharing or POS/counter error rather than a backend bug.
7. **Card/branch trace (`Sml.Sale`)**: pull `CustomerCardId`, `BranchOfficeId`, `PaymentTypeCode`, `PlatformCode` for every sale in the window. Identical `CustomerCardId` across disputed and undisputed sales rules out card cloning. Branches/platforms matching the member's own undisputed history rule out remote takeover — points toward shared card use or a counter-side error instead.

This path produces no hub/relay memory update — the member is the disputant, not a suspect, unless the trace itself surfaces a fraud pattern from the sections above.

---

### Points by period by EventTypeCode (reference query)

Useful for baselining normal distribution before anomaly investigation. Adjust the period granularity (`'yyyy-MM'` → `'yyyy-MM-dd'`) as needed.

```sql
SELECT
    FORMAT(DATEADD(HOUR, -3, LogDate), 'yyyy-MM')  AS Periodo_UTC3,
    EventTypeCode,
    COUNT(*)                                        AS Eventos,
    SUM(Points)                                     AS Puntos,
    AVG(Points)                                     AS Promedio,
    MAX(Points)                                     AS Maximo
FROM [SmartFran.Solution.SmartLoyalty].sml.CustomerPointsLog
WHERE
    LogDate >= '<inicio_GMT>'
    AND LogDate <  '<fin_GMT>'
GROUP BY
    FORMAT(DATEADD(HOUR, -3, LogDate), 'yyyy-MM'),
    EventTypeCode
ORDER BY Periodo_UTC3, Puntos DESC;
```

Expected distribution under normal conditions:

| EventTypeCode | Typical share of positive points |
|---|---|
| `EarnPointsByBuying` | ~85–95% |
| `EarnPointsByPromotion` | ~5–10% (0 pts each, high event count) |
| `CompensationalPoints` | < 5% (if no exploit) |
| `Article99999WithoutPoints` | 0 pts always |
| `DiscountPointsByExchange` | negative — separate analysis |

A session where `CompensationalPoints` represents > 20% of total positive points signals the exploit pattern. Cross-reference with `RegisterByUser` in `sml.ManualAssignPoints`.

---

### WHERE clause note — operator precedence

When filtering by two date ranges, always wrap the OR conditions in explicit parentheses:

```sql
WHERE
    (   (pt.Date >= '...' AND pt.Date < '...')
     OR (pt.Date >= '...' AND pt.Date < '...')
    )
```

Without the outer parentheses, `AND` binds before `OR` and any additional filter applies only to the second night. Filter is on `sml.PointsTransference.Date` — no `EventTypeCode` needed since every row in this table is a transfer.

---

## Name & Identity Flags

Apply these checks to every participant resolved in Q4 / identity resolution. Flag inline — do not defer.

### DNI validity (Southern Cone context)

**Argentina DNI structure (RENAPER):**

Argentina issues two separate DNI series:

| Population | Valid range | Notes |
|---|---|---|
| Native-born Argentines | 1,000,000 – ~65,000,000 | Sequential allocation; current issuance ~55–65M as of 2026 |
| Foreign legal residents (temporary or permanent) | Starts with **90, 93, 94, or 95** (millions) | Assigned by RENAPER upon obtaining residency; e.g., 90.000.001 – 90.999.999 |

| Country | Valid | Flag as invalid |
|---|---|---|
| Argentina (AR) | 1M–~65M (native), or 90M/93M/94M/95M series (foreign resident) | < 7 digits; < 1,000,000; sequential (123456); all-same-digit (18181818); in gap ranges 66M–89M, 91M–92M, 96M+ |
| Paraguay (PY) | 7–8 digits | < 7 digits or sequential |
| Uruguay (UY) | 7–8 digits | < 7 digits or sequential |
| Chile (CL) | RUT 7–9 digits + verifier | format mismatch |

**Invalid gap ranges (AR):** 66,000,000–89,999,999 (no DNI series assigned here), 91,000,000–92,999,999 (gap between foreign series), 96,000,000+ (above all known foreign series).

Additional automatic fake account flags:
- **Sequential**: `123456`, `1234567` — invalid in all countries.
- **All-same-digit / repeating pattern**: `18181818`, `24242424`, `11111111` — zero entropy, machine-generated.
- **Impossible range (AR)**: `341688840` (~341M), `92623449` (92M — falls in the 91–92M gap, not a valid native or foreign series).

> Eustacio C. Castro (DNI 92623449) is confirmed invalid — 92M is in the gap between the 90M and 93M foreign resident series.

### Name plausibility (Southern Cone context)

Flag any name that matches one or more of the following:

| Signal | Description |
|---|---|
| **Anagram / transposition** | Name shares all or most characters with another participant in the same session, reordered. Example: "Dimon Briz" ↔ "Simon Brizuela" — obvious alias. |
| **Phonetic near-match** | Name sounds like another participant's name when read aloud in Spanish. Example: "Dimon" → "Simon". |
| **Non-name character pattern** | Name is a number, a single letter, a keyboard sequence, or a generic placeholder ("Test", "Prueba", "Admin", "Cliente"). |
| **Inconsistent capitalization pattern** | All-caps or all-lowercase full name across accounts that otherwise have normal casing — signals bulk registration. |
| **Missing surname** | Single-token name with no apellido in a context where the field is required. |

### Known actor lookup

Before issuing any queries, read:
- `loyalty/memory/known_hubs.md`
- `loyalty/memory/known_relays.md`
- `loyalty/memory/actor_notes.md` — extended context; consult when an actor from the tables appears in the investigation

If any CustomerId or DNI from those files appears in the current investigation, prepend **[KNOWN HUB]** or **[KNOWN RELAY/FEEDER]** to that row in the analysis. Do not wait for query results to flag them.

### Updating memory files

When an investigation closes and an account is confirmed:
- **Hub/relay/feeder**: add row to `known_hubs.md` or `known_relays.md`. Columns (English): CustomerId, DNI, Name, Country, Role, Created, Confirmed, Events.
- **POS actor**: add row to `known_pos.md`. Columns (English): BranchOfficeId, Code, Branch, FranchiseGroupId, Franchise, Role, Confirmed, Events.
- **Extended context**: add a note block to `actor_notes.md` keyed by DNI (accounts) or Code (branches).

All memory files (`known_hubs.md`, `known_relays.md`, `known_pos.md`, `actor_notes.md`) are maintained in **English** for token efficiency.

> `sml.BranchOffice` SQL column is **`Code`** (not `Codigo`). Always write `bo.Code AS Codigo` in queries. "Codigo" is a display label only — using `bo.Codigo` directly will fail with error 207. Full schema: `Id, Name, Code, Description, FranchiseGroupId, AddressId, ActivatedDate, DeactivatedDate, DeactivateNote, CreatedDate, ExecutiveId`.

---

## Email Verification

**Schema finding (confirmed 2026-06-04):** No explicit email verification column exists anywhere in the schema. `sml.Person.Email` stores the address only. `SmlSt.CustomerMailing` includes a `CustomerId` column (confirmed by working Q_VALIDATE_EMAIL queries) plus geographic segmentation columns (LocationId, StateId, RegionId, StateName, RegionName, CityName). A NULL result on `LEFT JOIN SmlSt.CustomerMailing ON CustomerId` means the account is not registered in any mailing segment — the `NO_EN_MAILING` flag. `Mlg.MailContact.ContactData` is survey XML with no verification data.

The two available signals are:

### Signal 1 — Disposable email domain (HIGH confidence)

Check `sml.Person.Email` against known disposable/anonymous email providers. A disposable email on a loyalty account = synthetic account, no real identity link.

**Reference blocklist (community-maintained, thousands of domains):**
`https://raw.githubusercontent.com/disposable-email-domains/disposable-email-domains/refs/heads/main/disposable_email_blocklist.conf`

**Confirmed in investigations:**

| Domain | Confirmed on |
|---|---|
| `yopmail.com` | Simon Brizuela 46845173 + 46845174 (2026-06-04) |
| `hilostar.com` | Dimon Briz 123456 (2026-06-04) |
| `datehype.com` | Rafael Eduardo Colque 31035135 → Estefania Cazon (2026-06-12) |
| `bultoc.com` | Tuntuntuntun Sahur DNI 341688840 (2026-06-12) |

When a new disposable domain is found during an investigation, add it to this table.

### Signal 2 — Absence from `SmlSt.CustomerMailing` (MEDIUM confidence)

A `LEFT JOIN SmlSt.CustomerMailing ON CustomerId` returning NULL means the account is not part of any mailing campaign. On its own this is inconclusive (a legitimate customer may simply not be subscribed). Combined with Signal 1 or other fraud indicators, it confirms the synthetic pattern.

A real account will typically have a non-disposable email and be present in `CustomerMailing`. Absence alone is not sufficient — require at least one additional fraud indicator before flagging.

### Email check query — add to Q4 (identity resolution)

```sql
-- Agregar a Q4: email y señal de dominio desechable
LEFT JOIN [SmartFran.Solution.SmartLoyalty].SmlSt.CustomerMailing cm_s
    ON cm_s.CustomerId = sender.CustomerId
LEFT JOIN [SmartFran.Solution.SmartLoyalty].SmlSt.CustomerMailing cm_r
    ON cm_r.CustomerId = receiver.CustomerId
```

And project these columns:
```sql
    ps.Email                                                     AS Emisor_Email,
    CASE WHEN ps.Email LIKE '%yopmail.com'
          OR ps.Email LIKE '%hilostar.com'
          OR ps.Email LIKE '%mailinator.com'
          OR ps.Email LIKE '%guerrillamail.com'
          OR ps.Email LIKE '%tempmail.com'
         THEN 'DOMINIO_DESECHABLE'
         WHEN cm_s.CustomerId IS NULL THEN 'NO_EN_MAILING'
         ELSE 'OK'
    END                                                          AS Emisor_Email_Flag,
    pr.Email                                                     AS Receptor_Email,
    CASE WHEN pr.Email LIKE '%yopmail.com'
          OR pr.Email LIKE '%hilostar.com'
          OR pr.Email LIKE '%mailinator.com'
          OR pr.Email LIKE '%guerrillamail.com'
          OR pr.Email LIKE '%tempmail.com'
         THEN 'DOMINIO_DESECHABLE'
         WHEN cm_r.CustomerId IS NULL THEN 'NO_EN_MAILING'
         ELSE 'OK'
    END                                                          AS Receptor_Email_Flag,
```

---

## Fraud Patterns

| Pattern | Signal |
|---|---|
| **Point concentration (fan-in)** | Multiple senders → single `CustomerId` in a short window |
| **Automation cadence** | Transfers every 4–7 seconds for 30–40 min = scripted, not manual |
| **Limit breach** | Sender > 8,000 pts/day (transfer) or customer > 3,000 pts/day (accumulation) |
| **Credential stuffing signature** | Sender DNIs form a contiguous or near-contiguous numerical range = attacker processes a sorted credential dump in batches |
| **Rapid accumulate-and-transfer** | Account earns points then immediately re-sends them |
| **Circular transfer** | A→B then B→A within a short interval = points laundering |
| **Account consolidation** | Sender drains balance to 0, receiver balance far exceeds the transfer amount = points aggregated into one account ahead of a large redemption |
| **Dormant account activation** | Account with no prior activity suddenly initiates or receives large transfers |
| **Profile tampering** | `Person.UpdateDate` close to a transfer `LogDate` |
| **Duplicate accounts** | Multiple `Person.Id` sharing the same `UidCode` + `UidSerie` (scoped by `Country_Id`) |
| **Consecutive DNI same name** | Two accounts share the same full name with DNIs differing by 1 — same person, duplicate registration |
| **Fake identity** | DNI invalid for country (< 7 digits or sequential); name is an anagram/alias of another participant |
| **Unverified email on hub/sender** | Account email not verified at time of large transfer — synthetic account signal |
| **Promotion exploitation** | Many accounts earning from the same `PromotionId` in a short window |
| **Manual insertion** | `ManualAssignPointsId IS NOT NULL` — insider risk |
| **Uncapped manual injection** | Operator assigns > 3,000 pts in a single `ManualAssignPoints` grant; `Catalog_Id = NULL`; no approval gate — systemic if > 75% of grants exceed daily cap over multiple months |
| **Themed cluster registration** | Multiple accounts sharing a naming theme (meme characters, fictional universe, pop-culture references) combined with repeating-digit or impossible DNIs — signals a single operator performing coordinated synthetic registration. Confirmed pattern: Italian Brainrot characters (Tung Tung Sambayone / Tralalero Tralala / Tuntuntuntun Sahur, Jun 2026) with DNIs 18181818 / 51447556 / 341688840. |
| **One-shot welcome-bonus grab** | Account created → welcome bonus received → immediately redeemed (< 15 min) or immediately relayed (< 20 min). Account goes dormant. Target: the 5,000 pt `NewCustomer` grant. |
| **Deactivated account as sender** | A transfer where the sender's `sml.Person` record has all PII fields replaced with their own CustomerId GUID (confirmed deactivated) — indicates the transfer occurred before deactivation, OR a platform bug allows deactivated accounts to send. Always verify `sml.Customer.CreatedDate` vs transfer `LogDate` to determine sequence. |

---

## Evidence Package

### Transfer fraud

| File | Content |
|---|---|
| `01_queries_investigacion.sql` | All investigation queries (Q1–Qn) |
| `02_nocheN_transferencias.csv` | Identity + role + points per participant per night |
| `03_nocheN_emisores.csv` | Sender detail with limit validation |
| `04_pointslog_transferencias_YYYYMMDD.csv` | Raw `CustomerPointsLog` export via SSMS "Results to File" |
| `05_YYYYMMDD_transferencias_pm.csv` | PM-facing transfer detail: one row per transfer, includes names, DNIs, channel, plain-language fraud flags. See `sre-output` for column spec. |

Place all files under `events/YYYYMMDD_fraude_transferencias/`.

### Accumulation anomaly closure (`_ops.md`)

Required sections in `YYYYMMDD_description_ops.md`:

1. **Metrics table** — Eventos totales, Puntos totales, Ventana activa, Clientes únicos, Transacciones POS, Asignaciones manuales, Transferencias, Límites superados.
2. **EventTypeCode breakdown** — one row per code: EventTypeCode | Eventos | Puntos. Source: Q1.
3. **Customer detail** — one row per customer × EventTypeCode: Cliente | Documento | EventTypeCode | Transacciones | Puntos. Header specifies window in UTC-3. Source: Q4.

Place under `events/YYYYMMDD_description/`.

### Points accounting — required in every analysis output

Every investigation analysis (inline and in `_ops.md`) must cover two axes per actor:

#### Axis 1 — Account origin and identity validation (mandatory for every actor)

| Field | Source |
|---|---|
| Registration channel | `sml.Customer.RegistrationChannel` — `APP`, `WEB`, `PUNTO DE VENTA` |
| Registration date | `sml.Customer.CreatedDate` |
| Registering branch | `sml.Customer.RegisterById` → `sml.BranchOffice.Id` → `bo.Code`, `bo.Name`, franchise owner |
| Country | `sml.Customer.Country_Id` |
| **Email validation** | See Email Verification section — two signals: disposable domain (HIGH) + `SmlSt.CustomerMailing` absence (MEDIUM). No explicit verification column exists in the schema (confirmed 2026-06-04). Report both signals for every actor. |

Email validation query to include in identity resolution:
```sql
SELECT
    p.Email,
    CASE
        WHEN p.Email LIKE '%yopmail.com'
          OR p.Email LIKE '%hilostar.com'
          OR p.Email LIKE '%bultoc.com'
          OR p.Email LIKE '%datehype.com'
          OR p.Email LIKE '%mailinator.com'
          OR p.Email LIKE '%guerrillamail.com'
          OR p.Email LIKE '%tempmail.com'
        THEN 'DOMINIO_DESECHABLE'
        WHEN cm.CustomerId IS NULL
        THEN 'NO_EN_MAILING'
        ELSE 'OK'
    END AS Email_Flag
FROM sml.Person p
LEFT JOIN SmlSt.CustomerMailing cm ON cm.CustomerId = p.Id
WHERE p.Id = '<suspect_id>';
```

#### Axis 2 — Final point status (four states)

Replace "Recuperable / Irrecuperable" with the four statuses below. A single actor may contribute to multiple statuses.

| Status | Definition | Recovery |
|---|---|---|
| **Gastados** | Redeemed at POS via `DiscountPointsByExchange` — irreversible once ticket issued | None |
| **Transferidos** | Sent to another account via `PointsByTransferSent` — trace the downstream chain | Follow chain |
| **Activos** | Current balance in active (non-deactivated) account — `smlst.CustomerPointsLog.Points` | Suspend + reverse |
| **Retenidos** *(Held)* | Account deactivated (PII scrubbed in `sml.Person`) but `smlst.CustomerPointsLog` still has Points > 0 — account cannot transact but points exist and are admin-reversible by CustomerId | Admin reversal |

Detection query for **Retenidos**:
```sql
SELECT cst.CustomerId, cst.Points AS Pts_Retenidos, cst.LastLogDate
FROM smlst.CustomerPointsLog cst
JOIN sml.Person p ON p.Id = cst.CustomerId
WHERE cst.CustomerId = '<suspect_id>'
  AND TRY_CAST(p.UidSerie AS BIGINT) IS NULL   -- account deactivated
  AND cst.Points > 0;
```

#### Table format

| Actor | DNI | Rol | Origen (Canal / Fecha / Sucursal) | Pts Totales | Gastados | Transferidos | Activos | Retenidos |
|---|---|---|---|---|---|---|---|---|
| Tung Tung Sambayone | 18181818 | relay | APP / dic 2025 | 27.000 recv | ~14.000 | 38.000 | 2.830 | — |
| 5DB72B42 (desactivada) | scrubbed | feeder | POS 3822 / feb 2023 | 27.000 | 0 | 27.000 | — | (pendiente smlst) |

If a downstream account balance is unknown mark the cell `(pendiente)` and flag as time-sensitive — points may be cashed out while investigation is open.

> **Never assume point statuses from prior analysis or session memory.** Every cell in the accounting table must be backed by a query result from the current investigation. Use `smlst.CustomerPointsLog` to confirm current balances; use `sml.CustomerPointsLog` aggregated by `EventTypeCode` to confirm spent/transferred totals.

### Manual assignment exploit closure (`_ops.md`)

Same three required sections as accumulation anomaly, plus:

4. **Manual assignment detail table** — one row per grant: Fecha UTC-3 | Cliente | Documento | ManualAssignId | Puntos | Límite superado.
5. **Account balance status** — for each flagged account: CustomerId | Saldo actual | Puntos asignados | Reversión posible.
6. **Exploit scope section** — monthly trend table (Q_E4) + top beneficiaries (Q_E3) + total excess above daily cap (Q_E2).
7. **Action items** — split into: Inmediatas (reversals, operator suspension) / Técnicas (ceiling, Catalog_Id enforcement, approval workflow) / Auditoría histórica (redeemed vs intact, legal escalation).

### Member dispute closure (`_ops.md`)

Single-account scope — the three network-wide metrics (Clientes únicos, etc.) collapse to 1 and can be noted as such rather than omitted. Required sections:

1. **Metrics table** — same format as accumulation anomaly, scoped to one customer.
2. **EventTypeCode breakdown** — Q1-style, scoped to the account.
3. **Reconstrucción por transacción disputada** — one table per disputed date: hora UTC-3, SaleId, PromotionId, producto decodificado (`Sml.Promotion.Name`), puntos, sucursal, tarjeta, PaymentType. State explicitly whether each disputed line matches (same `SaleId`) or is separate from (different `SaleId`/product) any transaction the member acknowledges.
4. **Balance reconciliation table** — verified starting balance (direct `SUM`, not inferred) → running balance through every event → confirmed current balance. Must close exactly; note explicitly if it doesn't.
5. **Points accounting** — Axis 1 origin + Axis 2 four-state, same as any actor. For a dispute, the closure table (`Actor | DNI | Rol | Pts Totales | Recuperable | No Recuperable`) rol is "Socia — denunciante", not hub/relay/feeder.
6. **Acciones propuestas** — split into: verification steps with the branch/customer (not reversals yet) / conditional remediation (credit back if verification fails to find a mundane explanation) / explicitly note "Sin acciones técnicas" if no platform gap was found (distinguishes this from the exploit-closure format, where technical remediation is usually required).
