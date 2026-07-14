# Skill: Azure Operations

Investigation and remediation support for Azure infrastructure and Azure AD Domain Services (AADDS).

## Scope

- Azure AD Domain Services (AADDS) — health alerts, Kerberos policy, replica sets
- Azure VMs — compute state, diagnostics, extensions
- Azure Networking — NSGs, VNets, DNS
- Azure Monitor — alerts, metrics, activity log

For NSG-specific audits on `sfcgnetsec01`, use `/loyalty-azure-nsg`. For SmartFran Cloud Service Bus, multi-tenant App Services, and franchise onboarding diagnostics, use `/cloud-azure`.

## Known Infrastructure

| Resource | Value |
|---|---|
| Subscription / tenant | `smartit.azure` |
| Managed domain | `smartit.azure` (AADDS) |
| Replica set | `East US / sfcgvnet01 / DMZ-AD` |
| VNet | `sfcgvnet01` |
| Resource group (NSG) | `DefaultGroup01` |

## Command Constraint

> **Read commands** (`az ... show`, `az ... list`, `az ... describe`) — safe to run, no warning needed.
> **Write/action commands** (`az ... update`, `az ... set`, `az ... delete`, `az ... restart`, PowerShell `Set-*` / `Disable-*`) — always prefix the block with `⚠️ ACTION — this command modifies Azure state.`

---

## AADDS — Health and Alerts

```bash
# List all active AADDS alerts
az ad ds list --query "[].{Name:name, Health:healthLastEvaluated, Alerts:healthAlerts}" --output table

# Show full detail for a specific managed domain alert
az ad ds show --name smartit.azure --resource-group <resource-group> --output json
```

```bash
# Run Microsoft Entra Domain Services network diagnostics (read-only check)
# Requires AzureAD or Az.AD PowerShell module
Test-AzureADDSDomainService -Name smartit.azure
```

---

## AADDS — Kerberos RC4 Investigation

Identify accounts and objects with RC4 still enabled (`msDS-SupportedEncryptionTypes` not set to AES-only).

```powershell
# Find user accounts with RC4 encryption types (msDS-SupportedEncryptionTypes NOT restricted to AES)
# Run on a domain-joined machine or via AADDS PowerShell remoting
Get-ADUser -Filter * -Properties msDS-SupportedEncryptionTypes |
  Where-Object { $_."msDS-SupportedEncryptionTypes" -band 4 } |
  Select-Object SamAccountName, DistinguishedName, msDS-SupportedEncryptionTypes
```

```powershell
# Find computer accounts with RC4 enabled
Get-ADComputer -Filter * -Properties msDS-SupportedEncryptionTypes |
  Where-Object { $_."msDS-SupportedEncryptionTypes" -band 4 } |
  Select-Object Name, DistinguishedName, msDS-SupportedEncryptionTypes
```

```powershell
# Check GPO for Kerberos supported encryption types
# Look for: Network security: Configure encryption types allowed for Kerberos
Get-GPO -All | ForEach-Object {
  $report = Get-GPOReport -Guid $_.Id -ReportType Xml
  if ($report -match "KerberosSupportedEncryptionTypes") {
    Write-Output "$($_.DisplayName): $($_.Id)"
  }
}
```

**Bit reference for `msDS-SupportedEncryptionTypes`:**

| Bit | Value | Cipher |
|---|---|---|
| 0x01 | 1 | DES-CBC-CRC |
| 0x02 | 2 | DES-CBC-MD5 |
| 0x04 | 4 | RC4-HMAC |
| 0x08 | 8 | AES-128 |
| 0x10 | 16 | AES-256 |
| 0x18 | 24 | AES-128 + AES-256 (recommended) |

---

## AADDS — Kerberos RC4 Remediation

> ⚠️ ACTION — the commands below modify Azure / AD state. Test in a non-critical account first. Disabling RC4 before verifying AES support on all service accounts and computer accounts will break Kerberos authentication for those objects.

**Staged approach:**
1. Verify all service accounts and computer accounts have `msDS-SupportedEncryptionTypes` set to 24 (AES-128 + AES-256).
2. Update GPO to remove RC4 from allowed Kerberos encryption types.
3. Disable RC4 in AADDS managed domain settings.

```powershell
# Step 1 — Set AES-only on a specific service account
# ⚠️ ACTION — modifies AD attribute
Set-ADUser -Identity itservices -Replace @{"msDS-SupportedEncryptionTypes" = 24}
```

```powershell
# Step 1 — Set AES-only on a specific computer account
# ⚠️ ACTION — modifies AD attribute
Set-ADComputer -Identity SFCG-DB01 -Replace @{"msDS-SupportedEncryptionTypes" = 24}
```

```bash
# Step 3 — Disable RC4 at the AADDS managed domain level (via Azure CLI)
# ⚠️ ACTION — modifies AADDS managed domain Kerberos settings
az ad ds update \
  --name smartit.azure \
  --resource-group <resource-group> \
  --domain-security-settings kerberosRc4Encryption=Disabled
```

---

## Azure VMs — State and Diagnostics

```bash
# List all VMs with power state
az vm list -d --query "[].{Name:name, RG:resourceGroup, State:powerState, Size:hardwareProfile.vmSize}" --output table

# Show instance view (extensions, OS, health) for a specific VM
az vm get-instance-view --name <vm-name> --resource-group <resource-group> --output json

# List VM extensions installed on a host
az vm extension list --vm-name <vm-name> --resource-group <resource-group> --output table
```

---

## Azure Monitor — Action Group Email Warnings

> **Delivery delay:** Test notifications send sequentially per receiver and may show "Running" for 1–2 minutes per address. This is normal — wait up to 3 minutes before assuming a failure.

> **Email rate limit:** Azure Monitor caps email notifications at **100 emails per hour per receiver per region**. Additionally, an action group caps at **1,000 email actions total**. Once either limit is hit, alerts are silently suppressed. High-frequency alert rules (e.g. threshold firing every minute) can exhaust the hourly limit in minutes. Consider increasing the alert evaluation window or aggregation period to reduce firing frequency.

---

## Azure Monitor — Activity Log

```bash
# Show activity log for the last 24h (all operations on subscription)
az monitor activity-log list \
  --start-time "$(date -u -d '24 hours ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --output table \
  --query "[].{Time:eventTimestamp, Caller:caller, Operation:operationName.localizedValue, Status:status.value, Resource:resourceId}"

# Filter by resource group
az monitor activity-log list \
  --resource-group DefaultGroup01 \
  --start-time "$(date -u -d '24 hours ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --output table
```

---

## Constraints

- Default to read-only commands.
- Always warn with `⚠️ ACTION` before any command that modifies state.
- Never generate commands that delete resources or force-restart services unless explicitly requested.
- Output commands as copy-paste blocks. User runs them and pastes results back.
