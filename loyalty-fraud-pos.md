# Skill: Fraud Detection — POS & Franchise Networks

## Scope

Investigate point diversion via POS terminal manipulation in Club Grido franchise networks.

Triggered when: a transfer fraud investigation reveals an account with high-volume cross-branch POS activity, geographic impossibility, or direct report of franchise-level anomaly.

This skill operates on the **production database** (`SmartFran.Solution.SmartLoyalty`). Access is **implicit within this skill's scope** — no explicit user request required.

---

## Core Tables

### `sml.Sale`
`Id, SaleDate, BranchOfficeId, CustomerId, CustomerCardId, PointsLogId, InvalidatedPointsLogId, PaymentTypeCode, Distribution, PlatformCode`

One row per POS transaction. `CustomerId` = loyalty account scanned at the terminal.

| Column | Fraud relevance |
|---|---|
| `CustomerId` | Same CustomerId across hundreds of sales at multiple branches = hardcoded credential |
| `BranchOfficeId` | FK to `sml.BranchOffice` — identifies which branch processed the sale |
| `PlatformCode` | `POS` = in-store terminal |

### `sml.BranchOffice`
`Id, Name, Code, FranchiseGroupId, AddressId, ActivatedDate, DeactivatedDate, CreatedDate`

One row per franchise branch. FK to `sml.FranchiseGroup`.

### `sml.FranchiseGroup`
`Id, Name`

One row per franchise owner. A single owner may operate multiple branches.

### `sml.FranchiseStaff`
`Id, StaffId, FranchiseGroupId, StaffRoleCode, UserId, CreatedDate, DeactivatedDate`

Staff per franchise group. `StaffId` joins to `sml.Person.Id`. `DeactivatedDate IS NULL` = active.

| Signal | Description |
|---|---|
| Multiple staff sharing same email | Single insider controlling phantom staff identities |
| FranchiseManager + multiple lower roles under same email | Operator with elevated access masking behind subordinate accounts |

---

## Investigation Workflow

**Step 0:** Read `loyalty/memory/known_pos.md`. Flag any BranchOfficeId, Codigo, or Franquicia matching known actors before issuing queries.

### 1. Volume by branch

```sql
SELECT
    bo.Code, bo.Name AS Sucursal,
    fg.Name          AS Franquicia,
    COUNT(*)         AS Ventas,
    MIN(FORMAT(DATEADD(HOUR,-3,s.SaleDate),'yyyy-MM-dd')) AS Primera,
    MAX(FORMAT(DATEADD(HOUR,-3,s.SaleDate),'yyyy-MM-dd')) AS Ultima
FROM [SmartFran.Solution.SmartLoyalty].sml.Sale s
JOIN [SmartFran.Solution.SmartLoyalty].sml.BranchOffice bo ON bo.Id = s.BranchOfficeId
JOIN [SmartFran.Solution.SmartLoyalty].sml.FranchiseGroup fg ON fg.Id = bo.FranchiseGroupId
WHERE s.CustomerId = '<suspect_id>'
GROUP BY bo.Code, bo.Name, fg.Name
ORDER BY Ventas DESC;
```

Flag: same account in > 3 branches, especially across different provinces.

### 2. Geographic impossibility

Group sales by minute — same `CustomerId` in cities of different provinces within a few minutes = credential distributed across terminals, not a real person traveling.

### 3. Franchise owner mapping

Join `sml.BranchOffice → sml.FranchiseGroup` for all flagged branches. Branches sharing a `FranchiseGroupId` = same owner. Coordinated activity across owned branches is the primary insider signal.

### 4. Staff investigation

```sql
SELECT
    fg.Name         AS Franquicia,
    fs.StaffRoleCode,
    fs.CreatedDate,
    p.FirstName + ' ' + p.LastName AS Empleado,
    p.Email
FROM [SmartFran.Solution.SmartLoyalty].sml.FranchiseStaff fs
JOIN [SmartFran.Solution.SmartLoyalty].sml.FranchiseGroup fg ON fg.Id = fs.FranchiseGroupId
LEFT JOIN [SmartFran.Solution.SmartLoyalty].sml.Person p ON p.Id = fs.StaffId
WHERE fs.FranchiseGroupId IN ('<fg_id_1>', '<fg_id_2>')
  AND fs.DeactivatedDate IS NULL
ORDER BY fg.Name, fs.StaffRoleCode;
```

Flag: multiple staff rows sharing one email address.

### 5. Point yield by branch

```sql
SELECT
    bo.Code, bo.Name AS Sucursal,
    fg.Name          AS Franquicia,
    COUNT(DISTINCT s.Id)  AS Ventas,
    SUM(cpl.Points)       AS Puntos_Desviados
FROM [SmartFran.Solution.SmartLoyalty].sml.CustomerPointsLog cpl
JOIN [SmartFran.Solution.SmartLoyalty].sml.Sale s ON s.Id = cpl.SaleId
JOIN [SmartFran.Solution.SmartLoyalty].sml.BranchOffice bo ON bo.Id = s.BranchOfficeId
JOIN [SmartFran.Solution.SmartLoyalty].sml.FranchiseGroup fg ON fg.Id = bo.FranchiseGroupId
WHERE cpl.CustomerId = '<suspect_id>'
  AND cpl.EventTypeCode = 'EarnPointsByBuying'
  AND cpl.SaleId IS NOT NULL
GROUP BY bo.Code, bo.Name, fg.Name
ORDER BY Puntos_Desviados DESC;
```

Branches with high `Ventas` but 0 `Puntos_Desviados` = zero-point article terminals (less critical). Branches with real points = primary diversion source.

### 6. Redemption link

Query `DiscountPointsByExchange` on `sml.CustomerPointsLog` for the suspect account and any linked hub accounts. Join to `sml.Sale → sml.BranchOffice` to identify cashout branches. A single redemption branch appearing repeatedly = coordinated cashout point.

---

## Fraud Patterns

| Pattern | Signal |
|---|---|
| **POS credential hardcoded** | Same CustomerId across hundreds of sales at multiple branches / provinces |
| **Geographic impossibility** | Same account at geographically distant branches within minutes |
| **Multi-branch franchise coordination** | Same `FranchiseGroupId` across multiple high-volume branches with same diverted account |
| **Staff identity consolidation** | Multiple `FranchiseStaff` rows sharing one email = single insider with multiple roles |
| **Staged accumulation** | Account accumulates to a round target total, then executes a single large transfer — planned operation |
| **Accumulation/redemption split** | Points accumulated at franchise A, cashed out at franchise B — two-actor coordination |

---

## Memory Management

At investigation close, update `loyalty/memory/known_pos.md`:

**Accumulation branches table** — columns: `BranchOfficeId`, `Codigo`, `Sucursal`, `FranchiseGroupId`, `Franquicia`, `Pts desviados`, `Ventas`, `Confirmado`, `Eventos`.

**Redemption branches table** — columns: `BranchOfficeId`, `Codigo`, `Sucursal`, `FranchiseGroupId`, `Franquicia`, `SaleIds canje`, `Confirmado`, `Eventos`.

**Insider staff table** — columns: `Email`, `Nombre`, `Franquicia`, `Roles`, `Confirmado`.

---

## Evidence Package

| File | Content |
|---|---|
| `YYYYMMDD_consultas_pdv.sql` | All POS/franchise investigation queries |

Place under `events/YYYYMMDD_fraude_transferencias/` alongside the main investigation files.

Ops report must include (in addition to standard sections):
- Branch diversion table: Sucursal | Franquicia | Ventas | Puntos desviados
- Insider staff identified
- Acciones propuestas section including: terminal audit, staff access suspension, commercial proceedings, customer restitution
