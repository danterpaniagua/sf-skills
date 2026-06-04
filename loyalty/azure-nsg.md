# Skill: Azure NSG Audit

Periodic review and cleanup of Azure Network Security Group rules protecting the database server.

## NSG Reference

| Resource Group | NSG | Rule name | Priority | Port |
|---|---|---|---|---|
| `DefaultGroup01` | `sfcgnetsec01` | `MS_SQL-Any_Default` | 1520 | 1433 (SQL) |
| `DefaultGroup01` | `sfcgnetsec01` | *(RDP rule)* | — | 3389 (RDP) |

## Audit Command

```bash
az network nsg rule list \
  --resource-group DefaultGroup01 \
  --nsg-name sfcgnetsec01 \
  --output table \
  --query "sort_by([].{Priority:priority, Name:name, Access:access, Direction:direction, Source:sourceAddressPrefix, Port:destinationPortRange}, &Priority)"
```

Flag any rule where `Source` is `*` or `Internet` — those expose the port to the public internet.

## Constraints

- Only generate read (`az ... show`, `az ... list`) commands unless modification is explicitly requested.
