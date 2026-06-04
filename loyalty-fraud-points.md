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
| `CompensationalPoints` | Customer service compensation | Club Grido end customers | **Exploit confirmed** — no ceiling, no Catalog_Id |
| `HumanResourcesPoints` | Grido HR | **Grido employees only** — flag any non-employee | Legitimate — exclude from exploit totals |
| `InstitutionalPoints` | Institutional / internal | **Grido employees** (from / to) | Legitimate — exclude from exploit totals |
| `PrizePoints` | Prize / contest | End customers | Active 2023–2024; absent from 2025 — discontinuation unconfirmed |

**Exploit vector (confirmed 2026-06-02):** `sml.ManualAssignPoints` has no row-level ceiling, no mandatory `Catalog_Id`, and no approval workflow. A single operator can inject unlimited points to any customer with no technical control. Use Q_E1–Q_E5 to quantify historical exposure when this pattern is detected.

Auxiliary tables in the same schema (confirmed via `INFORMATION_SCHEMA`):
- `Sml.AssignmentPointsLog`
- `Sml.AssignmentPointsErrorLog`
- `Sml.ManualExchangeEnabled`
- `Sml.PreAssignedGiftRef`

---

### `sml.Customer`
`Id, FormDate, CreatedDate, UserId, EnrolledId, RegisterById, Country_Id, RegistrationChannel`

> **Note:** Column `DeactivatedDate` does not exist in this table (confirmed — SQL error `Invalid column name 'DeactivatedDate'`). Remove from any query targeting this table.

Relationship: `sml.CustomerPointsLog.CustomerId → sml.Customer.Id` (many-to-one).

| Column | Fraud relevance |
|---|---|
| `CreatedDate` | Cluster of accounts created in same window = coordinated bulk registration |
| `RegistrationChannel` | Same channel across many mule accounts = automated registration |
| `RegisterById` / `EnrolledId` | Same registrar across multiple accounts = insider or automated enroller |

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

| Country | Valid range | Flag if |
|---|---|---|
| Argentina (AR) | 7–8 digits, ≥ 1,000,000 | < 7 digits, or < 1,000,000, or sequential (1234, 12345, 123456) |
| Paraguay (PY) | 7–8 digits | < 7 digits or sequential |
| Uruguay (UY) | 7–8 digits | < 7 digits or sequential |
| Chile (CL) | RUT 7–9 digits + verifier | format mismatch |

A DNI of exactly `123456` (6 digits, sequential 1–6) is **invalid in all supported countries** — automatic fake account flag.

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
- **Hub/relay/feeder**: add row to `known_hubs.md` or `known_relays.md`. Columns: CustomerId, DNI, Nombre, País, Rol, Creado, Confirmado, Eventos.
- **POS actor**: add row to `known_pos.md`. Columns: BranchOfficeId, Codigo, Sucursal, FranchiseGroupId, Franquicia, Rol, Confirmado, Eventos.
- **Extended context**: add a note block to `actor_notes.md` keyed by DNI (accounts) or Codigo (branches).

---

## Email Verification

**Schema finding (confirmed 2026-06-04):** No explicit email verification column exists anywhere in the schema. `sml.Person.Email` stores the address only. `SmlSt.CustomerMailing` is a geographic mailing segmentation table (columns: LocationId, StateId, RegionId, StateName, RegionName, CityName) — `StateId` is province, not subscription status. `Mlg.MailContact.ContactData` is survey XML with no verification data.

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

### Manual assignment exploit closure (`_ops.md`)

Same three required sections as accumulation anomaly, plus:

4. **Manual assignment detail table** — one row per grant: Fecha UTC-3 | Cliente | Documento | ManualAssignId | Puntos | Límite superado.
5. **Account balance status** — for each flagged account: CustomerId | Saldo actual | Puntos asignados | Reversión posible.
6. **Exploit scope section** — monthly trend table (Q_E4) + top beneficiaries (Q_E3) + total excess above daily cap (Q_E2).
7. **Action items** — split into: Inmediatas (reversals, operator suspension) / Técnicas (ceiling, Catalog_Id enforcement, approval workflow) / Auditoría histórica (redeemed vs intact, legal escalation).
